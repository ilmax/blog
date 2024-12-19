---
title: "Kubernetes Network Debugging"
description: "Quick Kubernetes Networking Diagnosis: Learn how to spin up lightweight troubleshooting pods with just a few commands. Perfect for diagnosing cluster networking issues fast!"
date: 2024-12-07T13:33:58+01:00
draft: false
tags: ["Kubernetes", "network", "gist"]
---

If you need to do some network debuggin in Kubernetes, e.g. to verify DNS resolution, Firewall issues or something along these lines, you can spin up a pod with networking tools installed and have the pod removed when you leave the interactive shell (via the `--rm` argument). 

This can be achieved with the following commands:

```shell
kubectl run -it --rm dnsutil -n <namespace> --image=dnsutils -- /bin/bash
```

or

```shell
kubectl run -it --rm aks-ssh --namespace <namespace> --image=nicolaka/netshoot
```

This last one spins up a container with plenty of networking tools installed, you can find the list [here](https://github.com/nicolaka/netshoot).

I hope you find this useful!
