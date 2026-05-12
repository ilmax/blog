---
title: "K3s on Raspberry Pi — Moving the VIP outside the cluster with Keepalived"
description: "How I replaced MetalLB L2 with a dedicated external edge layer — two Radxa Zero 3E boards running keepalived (VRRP + IPVS NAT), forwarding directly to in-cluster Traefik NodePorts, fully provisioned with Ansible"
date: 2026-04-18T12:43:17+02:00
draft: false
series: ["K3s on Raspberry PI"]
series_order: 6
tags: ["Kubernetes", "homelab", "rpi", "ansible", "tutorial", "networking"]
---

## Background

When you install K3s on bare metal, something needs to sit in front of the cluster and hand out external IPs for `LoadBalancer` services. The standard answer for homelabs is [MetalLB](https://metallb.io/) in L2 mode, and it works — I ran it for months with no real problems.

But L2 mode has a structural limitation that bothered me: the load balancer lives *inside* the cluster. One elected node becomes the ARP speaker for each VIP, all inbound traffic funnels through that single node, and failover depends on ARP propagation which can take upward of ten seconds. In managed Kubernetes the load balancer is a separate piece of infrastructure. I wanted the same separation in my homelab

I had two [Radxa Zero 3E](https://radxa.com/products/zeros/zero3e/) boards sitting unused — small Rockchip RK3566 SBCs, one with 8 GB RAM and one with 4 GB. This is what I built with them.

## The old setup: MetalLB in L2 mode

MetalLB in L2 mode works by electing one node per `LoadBalancer` service as the *speaker*. That node announces the service VIP via gratuitous ARP, making itself the single entry point for all inbound traffic. Once traffic arrives at the speaker node, kube-proxy's IPVS rules distribute it to the actual pods — which may live on other nodes entirely.

{{<figure src="diagram-metallb.svg" alt="MetalLB L2 traffic flow" caption="*MetalLB L2: all inbound traffic flows through the elected speaker node before being distributed internally*" nozoom=true >}}

It works, but there are a few characteristics worth understanding before you decide whether it fits your needs:

- **Active-standby.** Only one node handles traffic per service at a time. The other nodes are idle from the ingress perspective.
- **The speaker is a bottleneck.** Every byte of inbound traffic passes through one node. If that node's NIC is saturated, you're saturated.
- **The load balancer lives inside the cluster.** MetalLB is a set of pods running in your cluster. If the cluster is degraded, so is your ingress path.

None of this is a bug — it's how L2 works. But for my homelab I wanted to implement something that more closely alignes to the manged Kubernetes model available in the cloud.

## The new architecture

The replacement is a dedicated edge layer built from two [Radxa Zero 3E](https://radxa.com/products/zeros/zero3e/) boards — compact ARM64 SBCs running DietPi. They sit on the same LAN as the Raspberry Pi cluster but are not part of it. The K3s installation explicitly disables the built-in ingress controller and service load balancer:

```yaml
# inventory.yml — master node install flags
k3s_install_command: >-
  /usr/local/bin/k3s-install.sh
    --disable traefik
    --disable servicelb
    ...
```

The cluster provides compute. The edge layer handles the VIP and load balancing. The end goal is an **active/active** setup using IPVS in **Direct Routing (DR)** mode — both edge nodes handling traffic simultaneously. This post covers the first step: moving the VIP outside the cluster with an active/passive NAT configuration.

{{<figure src="diagram-edge.svg" alt="Edge load balancer traffic flow" caption="*New architecture: Keepalived + IPVS NAT on the Radxa boards forms an external edge layer. Traffic is forwarded directly to in-cluster Traefik NodePorts on the K3s workers.*" nozoom=true >}}

Traffic flows like this:

1. A request arrives at the VIP (`192.168.20.127`), managed by **Keepalived** using VRRP
2. **IPVS** in NAT mode on the MASTER distributes packets across the K3s worker nodes via their NodePorts (`:30080` for HTTP, `:30443` for HTTPS)
3. The **in-cluster Traefik** receives the traffic and routes it to the appropriate pod
4. The response travels **back through the MASTER** — iptables MASQUERADE rewrites the source address so the worker nodes route replies through the edge

## Keepalived: VRRP and IPVS

Keepalived does two distinct jobs here: VIP management via VRRP, and load balancing via IPVS. They're configured in the same `keepalived.conf` but it helps to understand them separately.

### VRRP: the floating VIP

VRRP (Virtual Router Redundancy Protocol) is the mechanism that makes `192.168.20.127` float between the two load balancer nodes. One node is elected MASTER and holds the VIP; the other is BACKUP and monitors the MASTER via periodic advertisement packets. If advertisements stop arriving, the BACKUP promotes itself and takes over the VIP — typically in under a second.

The Ansible inventory sets the initial state for each node:

```yaml
# inventory.yml
loadbalancer_nodes:
  hosts:
    radxa-01:
      ansible_host: 192.168.20.6
      keepalived_state: MASTER
    radxa-02:
      ansible_host: 192.168.20.7
      keepalived_state: BACKUP
  vars:
    ansible_user: dietpi
    interface: eth0
    keepalived_virtual_server_ports:
      - { vip_port: 80,  node_port: 30080 }
      - { vip_port: 443, node_port: 30443 }

all:
  vars:
    vip_address: 192.168.20.127
```

The keepalived configuration template uses those variables:

```python
vrrp_instance VI_1 {
    state {{ keepalived_state }}        # initial state: MASTER or BACKUP
    interface {{ interface }}           # the NIC that carries VRRP traffic
    virtual_router_id 51                # group ID — must be identical on all nodes
    priority {{ '100' if keepalived_state == 'MASTER' else '90' }}  # higher wins election
    advert_int 0.2                      # advertisement every 200 ms; BACKUP promotes after ~600 ms silence
    unicast_src_ip {{ ansible_host | default(inventory_hostname) }}  # this node's IP
    unicast_peer {                      # send advertisement directly instead of multicast
{% for host in groups['loadbalancer_nodes'] %}
{% if host != inventory_hostname %}
        {{ hostvars[host].ansible_host | default(host) }}
{% endif %}
{% endfor %}
    }
    virtual_ipaddress {
        {{ vip_address }}               # the floating VIP — held by MASTER, taken over by BACKUP on failure
    }
    garp_master_delay 1                 # wait 1 s after becoming MASTER before sending gratuitous ARPs
    garp_master_repeat 3                # send 3 GARPs to flush switch/router ARP caches
    garp_master_refresh 2               # re-send GARPs every 2 s while MASTER
}
```

A few things worth noting. `priority` is derived from `keepalived_state` — 100 for MASTER, 90 for BACKUP — so the node that starts as MASTER will reclaim it after recovering from a failure (preemptive mode). The gap is deliberately small: both priorities are close together because there is no external health check weighting them down. `unicast_peer` is used instead of multicast: on a typical home network, unicast is more reliable and avoids any multicast filtering issues. `advert_int 0.2` means the BACKUP detects MASTER failure in under 600 ms — three missed advertisements at 200 ms each. The `garp_*` settings ensure that after a failover, the new MASTER sends multiple gratuitous ARPs to flush stale entries from switch and router ARP caches.

### IPVS: forwarding to the cluster

VRRP handles failover but it's not load balancing — only the MASTER holds the VIP. IPVS is what distributes traffic across the K3s worker nodes.

Keepalived manages IPVS virtual server entries that map each VIP port to the corresponding NodePort on the workers. The MASTER receives packets destined for the VIP and forwards them to the real servers (the K3s workers) using round-robin:

```python
{% for mapping in keepalived_virtual_server_ports %}
virtual_server {{ vip_address }} {{ mapping.vip_port }} {
    delay_loop 3             # health-check all real servers every 3 seconds
    lb_algo rr               # round-robin across real servers (several different algorithms can be configured)
    lb_kind NAT              # NAT mode: responses route back through the MASTER
    persistence_timeout 30   # pin a client to the same real server for 30 seconds
    protocol TCP

{% for host in groups['worker_nodes'] %}
    real_server {{ hostvars[host].ansible_host | default(host) }} {{ mapping.node_port }} {
        weight 1
        TCP_CHECK {
            connect_timeout 3        # seconds to wait for a TCP handshake
            retry 2                  # attempts before marking the server down
            delay_before_retry 2     # seconds between retries
        }
    }
{% endfor %}
}
{% endfor %}
```

{{< katex >}}

Each `virtual_server` block maps a VIP port to a NodePort — port 80 to 30080, port 443 to 30443. The real servers are the K3s workers, not the edge nodes themselves. `persistence_timeout 1` keeps connections pinned to the same worker for 30 seconds.

The `TCP_CHECK` parameters control how long it takes to pull a failed worker out of rotation. According to the [keepalived case study](https://keepalived.readthedocs.io/en/latest/case_study_healthcheck.html), a real server is considered down after the initial connection attempt fails, followed by `retry` additional attempts. Each attempt times out after `connect_timeout` seconds, with `delay_before_retry` seconds between them:

$$T_{removal} = \underbrace{connect\\_timeout}\_{\text{initial failure}} + retry \times (\underbrace{connect\\_timeout + delay\\_before\\_retry}\_{\text{each retry}})$$

With the current values: \\(T_{removal} = 3 + 2 \times (3 + 2) = 13\\) seconds.

`lb_kind NAT` means the MASTER rewrites both the destination address (on the way in) and the source address (on the way back). Unlike DSR, where responses bypass the load balancer entirely, NAT routes all return traffic through the MASTER.

This setup is **active/passive** — only the MASTER handles traffic at any given time (that's how the VRRP protocol works). Switching to DSR mode (`lb_kind DR`) would eliminate the return-path bottleneck by letting the workers respond directly to clients, but it requires additional configuration (loopback VIP alias and ARP suppression on each real server). True active/active — where both edge nodes handle traffic simultaneously — is also possible using two VIPs backed by two VRRP instances with inverted MASTER/BACKUP state on each node. I'll cover that in the next post.

NAT mode has simpler prerequisites than DSR — no loopback VIP alias is needed — but it requires IP forwarding, source NAT, and ARP tuning so that the worker nodes route their responses back through the edge. Here is the full install task for the keepalived role:

```yaml
---
# tasks/install.yml — install and configure keepalived
- name: "[install] Ensure keepalived group exists"
  ansible.builtin.group:
    name: keepalived
    state: present
    system: true

- name: "[install] Ensure keepalived user exists"
  ansible.builtin.user:
    name: keepalived
    group: keepalived
    system: true
    shell: /usr/sbin/nologin
    create_home: false

- name: "[install] Install keepalived and ipvsadm"
  ansible.builtin.apt:
    name:
      - keepalived
      - ipvsadm
    state: present
    update_cache: true

- name: "[install] Load IPVS kernel modules"
  ansible.builtin.command:
    cmd: modprobe {{ item }}
  loop:
    - ip_vs
    - ip_vs_rr
  changed_when: false

- name: "[install] Persist IPVS kernel modules across reboots"
  ansible.builtin.copy:
    dest: /etc/modules-load.d/ipvs.conf
    content: |
      ip_vs
      ip_vs_rr
    owner: root
    group: root
    mode: "0644"

- name: "[install] Configure sysctl for IPVS NAT and ARP"
  ansible.posix.sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    sysctl_set: true
    reload: true
  loop:
    - { name: "net.ipv4.ip_forward",                value: "1" }
    - { name: "net.ipv4.vs.conntrack",              value: "1" }
    - { name: "net.ipv4.conf.all.arp_ignore",       value: "1" }
    - { name: "net.ipv4.conf.all.arp_announce",     value: "2" }
    - { name: "net.ipv4.conf.default.arp_ignore",   value: "1" }
    - { name: "net.ipv4.conf.default.arp_announce", value: "2" }

- name: "[install] Install iptables-persistent for SNAT rules"
  ansible.builtin.apt:
    name:
      - iptables
      - iptables-persistent
    state: present
    update_cache: false

- name: "[install] Add MASQUERADE rule for IPVS NAT return traffic"
  ansible.builtin.iptables:
    table: nat
    chain: POSTROUTING
    destination: "{{ hostvars[item].ansible_host | default(item) }}"
    jump: MASQUERADE
    comment: "IPVS SNAT for {{ item }}"
  loop: "{{ groups['worker_nodes'] }}"
  notify: Persist iptables rules

- name: "[install] Configure keepalived"
  ansible.builtin.template:
    src: keepalived.conf.j2
    dest: /etc/keepalived/keepalived.conf
    owner: keepalived
    group: keepalived
    mode: "0644"
  notify: Restart keepalived

- name: "[install] Enable and start keepalived"
  ansible.builtin.systemd:
    name: keepalived
    enabled: true
    state: started
```

A few things worth highlighting. `ipvsadm` is the userspace tool that lets you inspect the IPVS table with `ipvsadm -Ln` — useful for debugging. The sysctl settings enable IP forwarding (`ip_forward`), IPVS connection tracking (`vs.conntrack`), and ARP tuning (`arp_ignore`, `arp_announce`) to prevent the edge from advertising the VIP on the wrong interface. The MASQUERADE rule is the key to NAT mode: without it, the worker nodes would try to reply directly to the client, but the client expects responses from the VIP, not from a worker IP. MASQUERADE rewrites the source address so the workers route replies back through the edge. The iptables changes are persisted via a handler triggered by `notify: Persist iptables rules`.

## Putting it together: the edge playbook

All of the above — keepalived, IPVS, iptables MASQUERADE — is provisioned by a single Ansible playbook: `playbooks/edge.yml`. It runs two plays in sequence:

```yaml
# Play 1 — collects facts from all K3s nodes
# Worker node IPs are needed to render the IPVS real_server list in keepalived.conf
- name: "[edge] Gather K3S cluster facts"
  hosts: k3s_cluster
  tags: [keepalived, edge]

# Play 2 — installs keepalived on both Radxas
# Configures VRRP, IPVS virtual servers, ip_forward, MASQUERADE
- name: "[keepalived] Install Keepalived on edge nodes"
  hosts: loadbalancer_nodes
  tags: [keepalived, edge]
```

The playbook uses tags as an operational affordance — you don't have to run everything every time:

```bash
# Install everything
ansible-playbook playbooks/edge.yml --tags edge

# Keepalived only (e.g. after changing the keepalived configuration)
ansible-playbook playbooks/edge.yml --tags keepalived
```

## In-cluster Traefik: the other side of the NodePort

The edge layer forwards raw TCP to the worker NodePorts, but the in-cluster Traefik still needs to know about the VIP. The Helm values configure Traefik as a `DaemonSet` with a `NodePort` service that advertises the VIP as its external address:

```yaml
# traefik/values.yaml (Ansible template)
ports:
  web:
    port: 80
    nodePort: 30080
  websecure:
    port: 443
    nodePort: 30443

service:
  enabled: true
  spec:
    type: NodePort
    externalIPs:
      - "{{ vip_address }}"
```

`externalIPs` tells Kubernetes to accept traffic addressed to the VIP on the NodePort service — without it, packets arriving from the edge via IPVS NAT would be dropped.

The Gateway API resource uses the same VIP for external DNS:

```yaml
gateway:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-dns01-production-issuer
    external-dns.alpha.kubernetes.io/target: "{{ vip_address }}"
  listeners:
    web:
      port: 80
      protocol: HTTP
      namespacePolicy:
        from: All
    websecure:
      namespacePolicy:
        from: All
      hostname: "*.myhomelab.domain"
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
      certificateRefs:
        - name: myhomelab.domain-tls
```

The `external-dns.alpha.kubernetes.io/target` annotation tells [external-dns](https://github.com/kubernetes-sigs/external-dns) to create DNS records pointing to the VIP. Combined with the `cert-manager` annotation, this means adding a new `HTTPRoute` that references this Gateway will automatically get a DNS record and a TLS certificate — no manual configuration on the edge.

## Seeing it in action

### One reboot, thirty timeouts

To get an honest measurement, I ran [hey](https://github.com/rakyll/hey) continuously and rebooted the master node mid-test. here's the result:

```shell
$ hey -z 50s -c 10 https://myhomelab.domain/

Summary:
  Total:        56.7313 secs
  Slowest:      4.9960 secs
  Fastest:      0.0089 secs
  Average:      0.0283 secs
  Requests/sec: 176.0226


Response time histogram:
  0.009 [1]     |
  0.508 [9945]  |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  1.006 [4]     |
  1.505 [0]     |
  2.004 [0]     |
  2.502 [0]     |
  3.001 [0]     |
  3.500 [0]     |
  3.999 [0]     |
  4.497 [0]     |
  4.996 [6]     |


Latency distribution:
  10%% in 0.0159 secs
  25%% in 0.0213 secs
  50%% in 0.0244 secs
  75%% in 0.0278 secs
  90%% in 0.0326 secs
  95%% in 0.0368 secs
  99%% in 0.0495 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0000 secs, 0.0444 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0117 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0017 secs
  resp wait:    0.0194 secs, 0.0062 secs, 0.1949 secs
  resp read:    0.0085 secs, 0.0002 secs, 4.9749 secs

Status code distribution:
  [200] 9956 responses

Error distribution:
  [30]  Get "https://myhomelab.domain/": dial tcp 192.168.20.127:443: connect: operation timed out
```

30 errors out of 9986 requests from a single MASTER reboot — a 0.3% failure rate. The errors are `operation timed out` rather than `connection refused` — this is the expected behavior with NAT mode. Since all return traffic routes through the MASTER, in-flight connections that were mid-response when it went down have no return path and must wait for TCP timeout. The 6 requests that hit ~5 s latency in the histogram are likely these stragglers completing just before the timeout threshold.

### Failover

While hey was running, I rebooted radxa-01. Here is what keepalived logged on radxa-02 (the BACKUP) during that same run:

```shell
dietpi@radxa-02:~$ sudo journalctl -u keepalived -f
May 04 11:23:31 radxa-02 Keepalived_healthcheckers[1202]: Activating healthchecker for service [192.168.20.3]:tcp:30080 for VS [192.168.20.127]:tcp:80
May 04 11:23:31 radxa-02 Keepalived_healthcheckers[1202]: Activating healthchecker for service [192.168.20.4]:tcp:30080 for VS [192.168.20.127]:tcp:80
May 04 11:23:31 radxa-02 Keepalived_healthcheckers[1202]: Activating healthchecker for service [192.168.20.5]:tcp:30080 for VS [192.168.20.127]:tcp:80
May 04 11:23:31 radxa-02 Keepalived_healthcheckers[1202]: Activating healthchecker for service [192.168.20.3]:tcp:30443 for VS [192.168.20.127]:tcp:443
May 04 11:23:31 radxa-02 Keepalived_healthcheckers[1202]: Activating healthchecker for service [192.168.20.4]:tcp:30443 for VS [192.168.20.127]:tcp:443
May 04 11:23:31 radxa-02 Keepalived_healthcheckers[1202]: Activating healthchecker for service [192.168.20.5]:tcp:30443 for VS [192.168.20.127]:tcp:443
May 04 11:23:31 radxa-02 Keepalived_vrrp[1203]: (VI_1) Entering BACKUP STATE (init)
May 04 11:29:19 radxa-02 Keepalived_vrrp[1203]: (VI_1) Entering MASTER STATE
May 04 11:29:45 radxa-02 Keepalived_vrrp[1203]: (VI_1) Master received advert from 192.168.20.6 with higher priority 100, ours 90
May 04 11:29:45 radxa-02 Keepalived_vrrp[1203]: (VI_1) Entering BACKUP STATE
```

The sequence:

1. **11:23:31** — radxa-02 starts in BACKUP state and activates health checks against all three K3s workers on their NodePorts. Note the real servers are now the workers (`.3`, `.4`, `.5`), not the edge nodes themselves.
2. **11:29:19** — radxa-01's VRRP advertisements stop. radxa-02 promotes itself to MASTER and takes over the VIP.
3. **11:29:45** — 26 seconds later, radxa-01 comes back with priority 100 (higher than radxa-02's 90) and reclaims MASTER. radxa-02 returns to BACKUP.

The VIP transfer itself is sub-second — three missed 200 ms advertisements. The 30 timeout errors in the hey output correspond to in-flight connections that were mid-response through radxa-01 when it went down. Because NAT routes responses back through the MASTER, those connections lost their return path and had to wait for TCP timeout.

## Dig deeper

- [keepalived documentation](https://keepalived.readthedocs.io/) — full reference for VRRP instance configuration and virtual server options
- [Linux IPVS sysctl reference](https://www.kernel.org/doc/html/latest/networking/ipvs-sysctl.html) — kernel documentation for all IPVS-related parameters
- [LVS NAT](http://www.linuxvirtualserver.org/VS-NAT.html) — the original LVS documentation on the NAT forwarding model
- [MetalLB L2 mode](https://metallb.universe.tf/concepts/layer2/) — the official explanation of how MetalLB L2 works, useful for understanding the tradeoffs discussed in this post

## Conclusion

The main goal of this post was to move the VIP outside the cluster. With MetalLB L2, the VIP lives on a Kubernetes node — if the cluster is unhealthy, so is your ingress path. By placing keepalived and IPVS on two dedicated Radxa boards, the VIP is now fully decoupled from K3s:

- The VIP lives outside the cluster, managed by infrastructure the cluster knows nothing about {{<iconc "check" "green">}}
- Sub-second failover via VRRP with gratuitous ARP flooding {{<iconc "check" "green">}}
- Full Ansible automation — a single two-play playbook provisions everything {{<iconc "check" "green">}}

The edge nodes are stateless L4 forwarders — keepalived handles the VIP and IPVS distributes traffic directly to the in-cluster Traefik NodePorts. The tradeoff is that this setup is still **active/passive**: only the MASTER handles traffic at any given time, and NAT routes all responses back through it.

In the next post, I'll turn this into an **active/active** setup — both edge nodes handling traffic simultaneously using IPVS in DSR mode with two VRRP instances and inverted priorities.

_Hero image generated by Bing Copilot_

Till the next time!
