---
title: "K8s Restart Pods on ConfigMaps Changes When Using Helm Charts"
date: 2025-04-16T13:54:42+02:00
draft: false
tags: ["Kubernetes", "helm", "gist"]
---

A small annoyance of using ConfigMaps in Kubernetes together with Helm charts is that a change in the ConfigMap does not trigger a pod restart. This is a well [known issue](https://github.com/kubernetes/kubernetes/issues/22368#issue-137938290), and there are even [some tools](https://github.com/stakater/Reloader) that aim to simplify this process.

Another, perhaps less well-known, solution to this problem is to include an annotation in the deployment, as suggested in the Helm documentation [here](https://helm.sh/docs/howto/charts_tips_and_tricks/#automatically-roll-deployments).

This is the example code:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        envFrom:
        - configMapRef:
            name: nginx-configmap
            optional: false
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-configmap
          optional: false
```

I hope you find this useful!
