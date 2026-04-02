---
title: "Flash Raspberry Pi OS over the network with Ansible"
description: "How to remotely reflash all Raspberry Pi nodes with a fresh OS using Ansible, overlayroot, and cloud-init — no KVM, no SD card puller needed"
date: 2026-04-02T09:00:00+02:00
draft: false
series: ["K3s on Raspberry PI"]
series_order: 5
tags: ["Kubernetes", "homelab", "rpi", "ansible", "tutorial"]
---

## Motivation

When I started building this homelab cluster I already had a several years of [Terraform](https://www.terraform.io/) under my belt, so infrastructure-as-code felt natural to me. Everything in the cloud gets declared in HCL, planned, and applied. I wanted the same level of automation for my Raspberry Pis.

The obvious tool for this is [Ansible](https://www.ansible.com/). Unlike Terraform, Ansible doesn't manage state files and doesn't model resources, it just runs tasks over SSH, which is exactly what you need for configuring Linux boxes. To get up to speed quickly I picked up [**Ansible for DevOps**](https://www.ansiblefordevops.com/) by Jeff Geerling. I can't recommend it highly enough, Jeff explains the concepts clearly and always gives practical, real-world examples.

The specific problem I wanted to solve: I got tired of flashing the OS manually. Every time I wanted to start fresh — to test a new Raspberry Pi OS release, to fix a wrong configuration, or just to rebuild the cluster cleanly I had to:

1. Pull each SD card (or NVMe drive) out of the Pi
2. Flash it with the Raspberry Pi Imager on my laptop
3. Put it back, power on, and wait

Multiply that by four nodes and it stops being fun. There had to be a better way to do this fully over the network, without touching any hardware.

This post explains how I automated the whole process.

> [!CAUTION]
> The playbook described in this article has only been tested on **Raspberry Pi 5 with 8GB RAM**. Some implementation details — like the RAM budget for holding the OS image in memory — are specific to this model. Running it on a Pi with less RAM or a different boot device layout may require adjustments.

## The clever bit: overlayroot

The trickiest part of remote OS flashing is the chicken-and-egg problem: the Pi is booted from the disk you want to overwrite. If you write to `/dev/sda` (or `/dev/nvme0n1`) while the OS is running from it, you corrupt the live filesystem.

The solution is a package called [overlayroot](https://packages.debian.org/sid/main/overlayroot). When enabled, it remounts the root filesystem as a **tmpfs overlay** — all writes go to RAM while the underlying disk is left untouched and mounted read-only as the lower layer. From the Pi's perspective the system still runs normally. From the flash perspective, the disk is free to overwrite.

```
Before overlayroot:           After overlayroot:
┌────────────────┐            ┌────────────────┐
│  / (ext4, rw)  │            │  / (overlay)   │ ← writes go here (RAM)
│  NVMe / SD     │            ├────────────────┤
└────────────────┘            │ upper (tmpfs)  │
                              ├────────────────┤
                              │ lower (ext4)   │ ← disk, now read-only
                              │  NVMe / SD     │   and safe to flash!
                              └────────────────┘
```

Once the Pi is in overlayroot, we can write a new OS image over the disk with `dd` while the system stays up long enough for Ansible to finish the task. The Pi then reboots cleanly into the new OS.

## The playbook structure

The playbook is split into six plays that run in sequence: a preflight check, a normalization step to ensure a clean starting state, pivoting root to RAM with overlayroot, downloading and preparing the OS image, flashing it with `dd`, and finally verifying the new OS came up correctly.

{{<figure src="flash-flow.svg" alt="Flowchart of the flash playbook steps" caption="*Flowchart of the Raspberry Pi OS flash process*" nozoom=true >}}

The playbook is also idempotent: at the start it reads `/etc/rpi-flash-version` (written by cloud-init on first boot) and compares it against the target image URL. If they match, all plays are skipped.

```yaml
# Check if the Pi is already running the target image
- name: Set already-flashed fact
  ansible.builtin.set_fact:
    rpi_already_flashed: >-
      {{
        _flash_ver_stat.stat.exists and
        (_flash_ver.content | b64decode | trim) == rpi_image_url
      }}
```

## Configuration

All node-specific variables live in the inventory. The `raspberrypis` group (which contains all K3s nodes) defines the target image URL and common OS settings:

```yaml
raspberrypis:
  hosts:
    pi-node-01:
      ansible_host: 192.168.20.1
    pi-node-02:
      ansible_host: 192.168.20.2
    pi-node-03:
      ansible_host: 192.168.20.3
    pi-node-04:
      ansible_host: 192.168.20.4
  vars:
    rpi_image_url: "https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2025-12-04/2025-12-04-raspios-trixie-arm64-lite.img.xz"
    rpi_user: pi-adm
    rpi_timezone: "Europe/Amsterdam"
    rpi_packages:
      - vim
      - curl
      - git
      - htop
      - python3-pip
```

The role defaults provide sensible fallback values for everything else:

```yaml
# roles/rpi-flash/defaults/main.yml
rpi_image_url: ""
rpi_image_path: "/tmp/{{ rpi_image_url | basename | regex_replace('\\.xz$', '') }}"

rpi_user: ""
rpi_timezone: ""
rpi_locale: "en_US.UTF-8"
rpi_keyboard_layout: "us"
rpi_ssh_public_key_path: "~/.ssh/id_ed25519.pub"
rpi_ssh_public_key: "{{ lookup('file', rpi_ssh_public_key_path) }}"
rpi_packages: []
rpi_dhcp: true
```

The SSH public key is read directly from your local `~/.ssh/id_ed25519.pub` using Ansible's `lookup('file', ...)`, so you never have to paste keys manually.

## The overlayroot play

This is where the magic happens. The play checks the current root filesystem type with `findmnt`. If the root is already an overlay (from a previous run), it skips the reboot and continues straight to the prepare phase. If not, it installs and configures `overlayroot`, then reboots:

```yaml
# tasks/overlayroot.yml

- name: Check current root filesystem type
  ansible.builtin.command: findmnt -n -o FSTYPE /
  register: _current_rootfs
  changed_when: false

- name: Install overlayroot package
  ansible.builtin.apt:
    name: overlayroot
    state: present
  when: _current_rootfs.stdout != "overlay"

- name: Configure overlayroot (tmpfs, swap enabled)
  ansible.builtin.lineinfile:
    path: /etc/overlayroot.conf
    regexp: '^overlayroot='
    line: 'overlayroot="tmpfs:swap=1,recurse=0"'
  when: _current_rootfs.stdout != "overlay"

- name: Reboot into overlayroot
  ansible.builtin.reboot:
    reboot_timeout: 120
    post_reboot_delay: 20
  when: _current_rootfs.stdout != "overlay"

- name: Fail if not running in overlay
  ansible.builtin.fail:
    msg: "Expected overlay filesystem at /, got '{{ _rootfs_type.stdout }}'"
  when: _rootfs_type.stdout != "overlay"
```

Note the `swap=1` option: with swap enabled, the overlay can spill over to the swapfile if RAM runs tight. This is important because we'll be holding the entire OS image in RAM during the prepare phase.

## Injecting cloud-init

Since Raspberry Pi OS is now based on Debian Trixie, it ships with [cloud-init](https://cloud-init.io/) out of the box. This is a big deal for automation: it means we can inject a configuration file into the OS image before flashing and have the Pi fully configure itself on first boot — user accounts, SSH keys, packages, timezone, network — without any manual steps.

The prepare play doesn't just copy the image — it modifies it before flashing. It mounts the FAT32 boot partition of the image file, then writes three cloud-init files into it:

- `user-data` — OS configuration (user, SSH key, packages, SSH hardening)
- `network-config` — network interface config (DHCP or static)
- `meta-data` — hostname for the NoCloud datasource

Here's the `user-data` template that gets rendered per-host:

```yaml
#cloud-config

hostname: {{ rpi_hostname | default(inventory_hostname) }}
manage_etc_hosts: true

users:
  - name: {{ rpi_user }}
    groups: [sudo, adm, dialout, gpio, i2c, spi]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    lock_passwd: true
    ssh_authorized_keys:
      - {{ rpi_ssh_public_key }}

ssh_pwauth: false

timezone: {{ rpi_timezone }}
locale: {{ rpi_locale }}

packages:
{% for pkg in rpi_packages %}
  - {{ pkg }}
{% endfor %}

package_update: true

write_files:
  - path: /etc/ssh/sshd_config.d/hardening.conf
    owner: root:root
    permissions: '0644'
    content: |
      PermitRootLogin no
      PasswordAuthentication no
      PubkeyAuthentication yes
      MaxAuthTries 3
      X11Forwarding no
```

When the Pi boots into the new OS for the first time, cloud-init reads these files, creates the user, installs packages, configures SSH, and sets the timezone — all without any manual intervention.

> [!NOTE]
> The Raspberry Pi OS uses the [NoCloud datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html) for cloud-init. It reads the config files directly from the FAT32 boot partition (the one mounted at `/boot/firmware`). You don't need a cloud provider or a metadata server.

## The flash play

Once the image is decompressed and cloud-init is injected, the flash play takes over. The first thing it does — before writing anything — is remove the Pi's old SSH host key from your local `known_hosts`. This is important: after flashing, the Pi will generate a new host key, and SSH would otherwise refuse to connect.

```yaml
- name: Remove old SSH host key locally
  ansible.builtin.command: ssh-keygen -R "{{ ansible_host }}"
  delegate_to: localhost
  become: false
```

Then it runs `dd`:

```yaml
- name: Write image to disk with dd
  ansible.builtin.shell: |
    dd if="{{ rpi_image_path }}" of="{{ flash_target }}" bs=4M conv=fsync status=progress
```

> [!WARNING]
> Writing the new image with `dd` corrupts the ext4 journal on the lower layer of the overlay filesystem. This can trigger a kernel panic and spontaneous reboot mid-task. The playbook handles this: the `reboot` task fires in the background (`sleep 2 && reboot &`) and Ansible waits for SSH to come back up on port 22 with a generous timeout. Both a clean reboot and a kernel-panic reboot are handled gracefully.

## Dig deeper

- [Jeff Geerling — Ansible for DevOps](https://www.ansiblefordevops.com/) — the book I used to learn Ansible, highly recommended if you're coming from a Terraform background
- [overlayroot.conf](https://sources.debian.org/src/cloud-initramfs-tools/0.18.debian8/overlayroot/etc/overlayroot.conf/) — configuration reference for the overlayroot tool
- [cloud-init NoCloud datasource](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html) — how cloud-init reads config from the boot partition without a metadata server
- [cloud-init user-data reference](https://cloudinit.readthedocs.io/en/latest/reference/examples.html) — all the things you can configure in user-data

## Conclusion

With this playbook, reflashing four Raspberry Pi nodes takes about as long as downloading the OS image once. No physical access, no SD card shuffling, no manual SSH key configuration. Just run the playbook, wait for the verify play to complete, and you're ready to reinstall K3s.

It took a bit of effort to understand overlayroot and get all the edge cases right (spontaneous kernel panics, boot device detection, cloud-init injection), but the result is an automation I'm genuinely happy to use every time I need to start fresh.

*Hero image generated by Bing Copilot*
