---
title: "Install K3s on a Raspberry PI - Master node"
description: Let's learn how to install a Kubernetes (k3s) multi-node cluster on Raspberry PI 5, master node setup
date: 2024-07-01T18:55:40+02:00
draft: false
series: ["K3s on Raspberry PI"]
series_order: 1
tags: ["Kubernetes", "homelab", "rpi", "tutorial"]
---

## Motivation

I recently bought a few Raspberry PI 5 SBCs to play around with Kubernetes at home and, without noticing, I started spending a lot of my free time with it.

The initial set of features that I wanted to set up were the following:

- Multinode k8s Cluster
- Do not expose any internal service to the internet
- Resolve internal services deployed on the cluster via domain name
- Have free SSL certificates and automated certificate renewal

As we will soon see, even this limited set of features, requires a decent amount of work to configure, so this process is split into a mini-series of articles to keep them relatively short.

> [!NOTE]
> Please note that I'm not going through all the O.S. installation and initial configuration in this article, so as a pre-requisite you have to install the O.S. into an SD card (or SSD if you happen to have the M2 Hat).

### Why K3s

One of the more common distributions of Kubernetes that are installed on the Raspberry PI is K3s. K3s is a **certified** and **lightweight** distribution, it's also a goal of the project **not to diverge** from the main Kubernetes code, which makes it perfect for getting your hands dirty with Kubernetes on resource-constrained devices.

## Operating System Preparation

### OS & Packages updates

Let's first update all the installed packages and the kernel itself.

1. SSH into the Raspberry PI using either the `ssh user@hostname.domain` format (if you don't have a domain configured you can use the hostname.local, for me for example it's `ssh pi-node-01.local`) or `ssh user@ip-address`
1. Update apt packages & OS using the following commands:

```sh
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo apt --fix-broken install -y
sudo apt autoremove -y
sudo apt autoclean
```

> [!NOTE]
> More information on what is the **local** domain and how it works can be found [here](https://en.wikipedia.org/wiki/.local)

### Configure a static IP on the Raspberry PI

I decided to set each Raspberry PI IP to a static value, this step is not strictly required but if your node changes IP, you will be in for a fun debugging session, so just to make things simpler, let's configure the Raspberry PIs with a static IP.
You can choose to do so in 2 different ways:

1. Configure the router's DHCP reservation (if available) so that the router assigns the same IP address to the device identified by its physical MAC address
1. Configure the router to limit the DHCP addresses available, and set the Raspberry PI IP configuration to manual

I chose the second option because my router doesn't support DHCP reservation, so here's what I've done:

1. Configure the router to limit the DHCP addresses to the range 192.168.2.1 to 192.168.2.200
1. Configure each Raspberry PI with an address starting from 201 up with the command below:

```sh
sudo nmcli con mod preconfigured ipv4.method manual ipv4.addr 192.168.2.201/24 ipv4.gateway 192.168.2.254 "192.168.2.59 1.1.1.1"
sudo reboot
```

>Please note that 192.168.2.59 is the IP of my Pi-Hole used as my dns resolver

where **preconfigured** is the name of my connection, configured by the Raspberry PI Imager that I preconfigure while flashing the O.S. to the SD card.
You can list all the connection names using the command `nmcli connections show`.
If you're using the WiFi interface, you can get the name of the connection using `nmcli connection show | grep wifi | cut -d' ' -f1`

### Configure the kernel to enable cgroup v2

K3s requires cgroup v2 to function and, by default, they're disabled in the Raspberry PI O.S. so we need to enable them as follows:

```sh
echo ' cgroup_memory=1 cgroup_enable=memory' | sudo tee -a /boot/firmware/cmdline.txt
sudo reboot
```

> [!WARNING]
> Without memory cgroup v2 enabled, k3s will fail to start with the following error: `level=fatal msg="failed to find memory cgroup (v2)"`

### Verify cgroup v2 is enabled

To verify that cgroup v2 is enabled we can run this command on the Raspberry PI:

```sh
pi-adm@pi-node-01:~$ grep cgroup /proc/filesystems

nodev   cgroup
nodev   cgroup2 <-- This tells us that cgroup v2 is enabled
```

## Master Node Installation

The first node that we will install is the master node, I'm calling this out because the install instruction is a little different between the master node and the worker nodes.

1. SSH into your Raspberry PI using `ssh user@hostname.domain` or `ssh user@ip-address`
1. Generate a random token, e.g. using OpenSSL

    ```sh
    export K3S_TOKEN=$(openssl rand -base64 20)
    ```

1. Prepare the K3s kubelet configuration file in `/etc/rancher/k3s/kubelet.config`

    ```yml
    kubeconfig=/etc/rancher/k3s/kubelet.config
    sudo mkdir -p $(dirname $kubeconfig)
    sudo tee $kubeconfig >/dev/null <<EOF
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    shutdownGracePeriod: 30s
    shutdownGracePeriodCriticalPods: 10s
    EOF
    ```

    This kubelet configuration enables the new kubernetes feature [Graceful Shutdown](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#graceful-node-shutdown) ensuring that the pod follows the normal [pod termination process](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) during the node shutdown.

1. Install the K3s control plane node

```sh
curl -sfL https://get.k3s.io | K3S_TOKEN=$K3S_TOKEN sh -s - server \
  --write-kubeconfig-mode '0644' --node-taint 'node-role.kubernetes.io/control-plane:NoSchedule' \
  --disable 'servicelb' --disable 'traefik' \
  --kube-controller-manager-arg 'bind-address=0.0.0.0' --kube-proxy-arg 'metrics-bind-address=0.0.0.0' \
  --kube-scheduler-arg 'bind-address=0.0.0.0' --kubelet-arg 'config=/etc/rancher/k3s/kubelet.config' \
  --kube-controller-manager-arg 'terminated-pod-gc-threshold=10'
```

> [!NOTE]
> Please note that the argument `--node-taint...` tells Kubernetes not to schedule pods on this node, if you want to schedule pods on the master node as well, remove that argument but bear in mind that's not the suggested approach. If you set up a single Kubernetes node though, you have to remove that argument.

### Installation parameters

Let's now look at all the parameters that we specified in the command line:

- `-s server` Used to tell K3s to run in server mode (for master node) as opposed to agent mode (for worker nodes)
- `--write-kubeconfig-mode '0644'` Writes the kubeconfig file with the specified mode
- `--node-taint 'node-role.kubernetes.io/control-plane:NoSchedule'` Tells K3s to not schedule any user pods on the master node, K3s common services: core-dns and metric-service will still run on the master node
- `--disable 'servicelb'` Do not install the built-in service load balancer (we will replace it with MetalLb later on)
- `--disable 'traefik'` Do not install Traefik, we will install it manually so we will have access to its configuration
- `--kube-controller-manager-arg 'bind-address=0.0.0.0'` Bind on all addresses to enable metrics scraping from an external node
- `--kube-proxy-arg 'metrics-bind-address=0.0.0.0'` Bind on all addresses to enable metrics scraping from an external node
- `--kube-scheduler-arg 'bind-address=0.0.0.0'` Bind on all addresses to enable metrics scraping from an external node
- `--kubelet-arg 'config=/etc/rancher/k3s/kubelet.config'` Specify the location of the kubelet config file (the one we generated in the previous step)
- `--kube-controller-manager-arg 'terminated-pod-gc-threshold=3'` This setting limits to 3 the number of terminated pods that can exist before the terminated pod garbage collector starts deleting terminated pods. See [Pod Garbage collection](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)

### Copy the kubeconfig files

To be able to connect to the cluster via the kubectl CLI, we need to copy the kubeconfig file in the default directory where the kubectl expects it:

```sh
 mkdir ~/.kube
 cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

### Verify master node installation

If everything is configured correctly, from within the node we can check the status of the nodes as shown below:

```sh
kubectl get nodes
NAME                  STATUS   ROLES                  AGE    VERSION
pi-node-01            Ready    control-plane,master   20s   v1.29.5+k3s1
```

> [!NOTE]
> To troubleshoot the installation you can look at the logs of the k3s service with `journalctl -u k3s` or `journalctl -xeu k3s.service`

## MetalLB Installation

We will install MetalLB as our load balancer controller to enable external access to cluster services. In a cloud scenario, the managed Kubernetes offering comes with a load balancer that gives you the ability to expose services of type `LoadBalancer` using a public IP (On Azure for example you can use a public [Standard Load Balancer](https://learn.microsoft.com/en-us/azure/aks/load-balancer-standard)).

K3s comes with a built-in load balancer called serviceLB, but this only exposes services on the node addresses, with something like MetalLB we can instead use a service of type `LoadBalancer` and get a specific IP for that service.

Without MetalLB if you create a service of type `LoadBalancer`, the external IP will be stuck in `<pending>` state when looking at it via the `kubectl get services -n namespace`.

In our case, we need to expose the Traefik Ingress Controller using an IP address of the internal home network and MetalLB allows us to achieve this.

MetalLB can be configured in two different modes:

- Layer 2 mode (where a single node gets all the traffic for a given service IP and then kube-proxy redirects to the service's pods)
- BGP mode (where each node establishes a BGP peering session with the network router and uses the peering session to advertise the IPs of external cluster services)

Both approaches have their pros and cons, in the layer 2 case, a single node can become a bottleneck while using BGP, if a node goes down, all active connections to the service will be terminated.
Since there's no clear winner here, I'm opting for using the layer 2 approach.

> [!WARNING]
> You can look at the limitation for [layer 2 mode](https://metallb.universe.tf/concepts/layer2/#limitations) or the [BGP mode](https://metallb.universe.tf/concepts/bgp/#limitations) on the MetalLB documentation

Logically we need to reserve some IP in the DHCP configuration so that there won't be multiple appliances with the same IP on the network. You should be able to configure your router to reserve some IP addresses, and then you can use those IP addresses in a Metallb `AddressPool`.

1. Install Metallb via kubectl apply

    ```sh
    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
    ```

1. Create a file called `metallb.yml` and place the following content there:

   ```yml
   ---
   apiVersion: metallb.io/v1beta1
   kind: IPAddressPool
   metadata:
     name: k3s-lb-pool
     namespace: metallb-system
   spec:
     addresses:
     ## Replace this with your IP address reserved range.
     ## This should be on the same network as your nodes!
     ## This address range should be blocked on your router DHCP config
     - 192.168.2.210-192.168.2.230
   ---
   apiVersion: metallb.io/v1beta1
   kind: L2Advertisement
   metadata:
     name: k3s-lb-pool
     namespace: metallb-system
   ```

### Configuring the WiFI interface (Optional)

**If you connect the Raspberry PI to the network via ethernet, skip this step.**

If you connect the Raspberry PI to the network via WiFi, you need to also change the configuration of the WiFi interface to make sure MetalLB works properly, you can see this in the product [documentation](https://metallb.universe.tf/troubleshooting/#using-wifi-and-can-t-reach-the-service).

> You can configure the interface (most likely called wlan0) with the following command: `sudo ifconfig <device> promisc`

> [!WARNING]
> **Bear in mind that this configuration doesn't survive the node restart**

### Make WiFi interface configuration persistent (Optional)

**If you connect the Raspberry PI to the network via ethernet, skip this step.**

If you want to make the configuration persistent, we need to implement a little workaround using a Linux service that starts after the network service comes online, and executes our command `ifconfig wlan0 promisc`

Here below you can see the code for such a service:

```sh
sudo bash -c 'cat > /etc/systemd/system/bridge-promisc.service' <<EOS
[Unit]
Description=Makes interfaces run in promiscuous mode at boot
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ifconfig wlan0 promisc
TimeoutStartSec=0
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOS

sudo systemctl enable bridge-promisc
```

### Verify MetalLB installation

To ensure the successful installation of MetalLB we can deploy a temporary service (for example an nginx image) and make sure it successfully gets assigned an external IP address.

```sh
kubectl create namespace test-metallb
kubectl create deployment nginx --image=nginx -n test-metallb
kubectl expose deployment nginx --type=LoadBalancer --name=nginx --port=80 --protocol=TCP -n test-metallb
```

After the deployment succeeds, we need to check if the newly created service gets an external IP address via:

```sh
kubectl get service -n test-metallb
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.43.247.187   192.168.2.210   80:30129/TCP   13s
```

After navigating to the exposed IP address in the browser, we can now safely remove this test namespace via:

```sh
kubectl delete namespace test-metallb
```

## Traefik installation

I picked Traefik as my Kubernetes Ingress Controller. This comes built into the K3s bundle but I decided to install it manually to target the latest version available (3.10 at the time of writing) hence I disabled it while installing K3s so, here we are, installing it manually.

We can use Traefik on Kubernetes in different ways:

- Using standard Kubernetes Ingress (May require several annotations for non-straightforward configurations)
- Using Traefik IngressRoute (Custom CRD)
- Using the newer Kubernetes Gateway API (Support of the Gateway API is not yet complete at the time of writing, July 2024)

I decided to use the custom CRD approach waiting for the GA support of the Gateway API (a new and improved networking specification that is the successor of the Ingress) because of the little configuration required and it also resonates well with me.

Anyway let's see how to install and configure Traefik:

1. Add Traefik Helm repositories & update them

    ```sh
    helm repo add traefik https://traefik.github.io/charts
    helm repo update
    ```

1. Create a values file called `traefikvalues.yml` where we specify the helm chart configuration values with the following content:

    ```yml
    deployment:
      enabled: true
      replicas: 2

    ports:
      web:
        redirectTo:
          port: websecure
      websecure:
        http3:
          enabled: true
        advertisedPort: 4443
        tls:
          enabled: true

    ingressRoute:
      dashboard:
        enabled: false

    providers:
      kubernetesCRD:
        enabled: true
        allowExternalNameServices: true
      kubernetesIngress:
        enabled: true
        allowExternalNameServices: true
        publishedService:
          enabled: false

    rbac:
      enabled: true

    service:
      enabled: true
      type: LoadBalancer
    ```

1. Run the following command to install Traefik:

    ```sh
    helm install --namespace=traefik traefik traefik/traefik --values=traefikvalues.yml --create-namespace
    ```

### Verify Traefik installation

1. Verify Traefik is installed and gets an external IP address

    ```sh
    kubectl get svc -n traefik
    NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                    AGE
    traefik   LoadBalancer   10.43.109.247   192.168.2.210   80:31288/TCP,443:32133/TCP,443:32133/UDP   30s
    ```

## Configure kubectl access on your machine

So far we always accessed the cluster from within the Raspberry PI node, so it's now time to configure our machine to access the cluster via kubectl, to do so we need to do two things:

1. Copy the kubeconfig file from the Raspberry PI to our machine and put it in a specific location, we can do so with `scp` as follows:

    ```sh
    mkdir ~/.kube
    scp user@hostname:/home/user/.kube/config ~/.kube/config
    ```

    replacing user with the user you configured on the Raspberry PI and hostname with the hostname set on the Raspberry PI. To me, the actual command looks like:

    ```sh
    mkdir ~/.kube
    scp pi-adm@pi-node-01.local:/home/pi-adm/.kube/config ~/.kube/config
    ```

1. Change the server address in the kubeconfig file we just copied, with the actual IP address of the cluster master node, in my case, it is 192.168.2.201

### Verify kubectl can access the cluster

Having done this, we should be able to successfully connect to the cluster. We can verify everything works correctly by executing any kubectl command, for example, `kubectl get nodes`

## Conclusion

At this point we have a basic K3s cluster up & running, we're in a good place to start adding additional nodes to the cluster and implementing the features mentioned at the beginning of the post.

If you enjoyed reading this far, stay tuned for the upcoming one on configuring the additional nodes.

*Hero Image generated by Bing Copilot*
