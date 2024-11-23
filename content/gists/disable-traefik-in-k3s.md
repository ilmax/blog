---
title: "Disable bundled Traefik in K3S"
description: How to make sure you disable traefik when installing K3S
date: 2024-11-23T09:50:02+01:00
draft: false
tags: ["Kubernetes", "homelab", "rpi", "gist"]
---

While automating the installtion of my K3S cluster with ansible, I struggled to disable traefik (and I'm not the only one, see [this](https://github.com/k3s-io/k3s/issues/1160)), the issue what that the file `/etc/systemd/system/k3s.service` was looking like this:

```service
ExecStart=/usr/local/bin/k3s \
    server \
        '--write-kubeconfig-mode' \
        ' 0644' \
        '--node-taint' \
        'node-role.kubernetes.io/control-plane:NoSchedule' \
        ' --disable' \
        'traefik' \
        ' --disable' \
        'servicelb' \
        ' --disable' \
        'local-storage' \
        ' --kubelet-arg' \
        'config=/etc/rancher/k3s/kubelet.config' \
        ' --kube-controller-manager-arg' \
        'bind-address=0.0.0.0' \
        ' --kube-proxy-arg' \
        'metrics-bind-address=0.0.0.0' \
        ' --kube-scheduler-arg' \
        'bind-address=0.0.0.0' \
        ' --kube-controller-manager-arg' \
        'terminated-pod-gc-threshold=10' \
```

This version still ends up installing Traefik v2, bundled with K3S.
The correct `k3s.service` file look like the following:

```service
ExecStart=/usr/local/bin/k3s \
    server \
        '--write-kubeconfig-mode' \
        '0644' \
        '--node-taint' \
        'node-role.kubernetes.io/control-plane:NoSchedule' \
        '--disable' \
        'traefik' \
        '--disable' \
        'servicelb' \
        '--disable' \
        'local-storage' \
        '--kubelet-arg' \
        'config=/etc/rancher/k3s/kubelet.config' \
        '--kube-controller-manager-arg' \
        'bind-address=0.0.0.0' \
        '--kube-proxy-arg' \
        'metrics-bind-address=0.0.0.0' \
        '--kube-scheduler-arg' \
        'bind-address=0.0.0.0' \
        '--kube-controller-manager-arg' \
        'terminated-pod-gc-threshold=10' \
```

The fix was to configure yaml to account for end of line properly, check [this](https://yaml-multiline.info/) to dig deeper into yaml multiline options.
