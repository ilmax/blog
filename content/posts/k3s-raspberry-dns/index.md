---
title: "Install K3s on a Raspberry PI - Automatic external DNS management"
description: Let's learn how to use ExternalDNS to automate DNS record management with CloudFlare or any other DNS provider in Kubernetes
date: 2024-08-05T21:11:10+02:00
draft: false
series: ["K3s on Raspberry PI"]
series_order: 4
tags: ["Kubernetes", "homelab", "rpi", "tutorial"]
---
Can we automate DNS management in Kubernetes? Yes we can, if you would like to know more, follow along to discover what ExternalDNS can do to make our lives easier.

ExternalDNS is used to automate DNS management, it essentially monitors some Kubernetes resources that are then used to build DNS records and finally, it synchronizes the changes with the DNS provider.

Its goal is defined as:

{{<lead>}}
ExternalDNS synchronizes exposed Kubernetes Services and Ingresses with DNS providers
{{</lead>}}

Automating DNS management helps to make sure we don't have to manually create DNS entries whenever we deploy a new service or that we don't leave [dangling DNS records](https://www.paloaltonetworks.com/cyberpedia/what-is-a-dangling-dns) whenever we delete an exposed service.

ExternalDNS offers two key benefits: it simplifies the deployment of new services in Kubernetes by automatically creating the necessary DNS records based on the provided annotations, and it ensures that any unused records are promptly cleaned up.

ExternalDNS achieves this by watching some resources (e.g. `Ingress`, `Service`, `IngressRoute`, etc) it then generates the corresponding DNS records and it takes care of keeping our DNS provider in sync, simple enough right?

To get ExternalDNS working took me way more time than I want to admit, so I decided to document how I configured ExternalDNS and the result I got. Without further ado, let's dive into it.

## Concepts

ExternalDNS generates DNS records from exposed Kubernetes **sources** that will then be synchronized with DNS **providers**.

{{<figure src="flowchart.svg" alt="ExternalDNS conceptual processing flow" caption="*Simplified flowchart of ExternalDNS*">}}

### Sources

ExternalDNS can monitor several different types of Kubernetes resources used to expose an application, among the supported ones we can find `Ingress`, `Services`, Gateway's `HTTPRoute` and also Traefik's own `IngressRoute` (the one I used in the preceding blog posts to expose services).

A source represents a single Kubernetes resource type that ExternalDNS watches for changes and is then used to construct the DNS records that will be synced in the DNS provider. The sources that ExternalDNS monitors are configurable, and it supports multiple types of sources.

{{<tip>}}
The list of supported sources can be found in the [documentation](https://kubernetes-sigs.github.io/external-dns/v0.14.2/sources/sources/).
{{</tip>}}

### Providers

A provider identifies the external service where your DNS zone lives, that's where ExternalDNS synchronizes the DNS records.
There are a lot of supported providers: Google Cloud DNS, AWS Route 53, AzureDNS, CloudFlare, GoDaddy, and even Pi-hole just to name a few.

Each provider implementation comes with its stability levels, the one I'm interested in, CloudFlare, has a **beta** stability level which is defined as:

> **Beta**: Community supported, well tested, but maintainers have no access to resources to execute integration tests on the real platform and/or are not using it in production.

So far during my testing, the CloudFlare provider worked flawlessly for both additions and deletions.

As is the case for the sources, the provider is also configurable.

{{<tip>}}
The list of supported providers and their respective stability levels can be found in the [documentation](https://kubernetes-sigs.github.io/external-dns/v0.14.2/#status-of-in-tree-providers).
{{</tip>}}

### Annotations

ExternalDNS comes with a slew of annotations that allow you to customize its behavior for every single service, some of the most used are probably:

- **external-dns.alpha.kubernetes.io/ttl** that specifies the time to live (TTL) of the DNS record
- **external-dns.alpha.kubernetes.io/target** that specifies the DNS record targets (Its usage will be covered in the [Recipes](#recipes) section)

{{<tip>}}
The list of supported annotations can be found in the [documentation](https://kubernetes-sigs.github.io/external-dns/v0.14.2/annotations/annotations/).
{{</tip>}}

### Record Ownership

To make sure ExternalDNS won't mess up the DNS records that already exist in the DNS zone, an ownership concept is implemented. This ownership concept marks all records that ExternalDNS creates, this way it knows it has full control over those records and it will simple ignore all the other records that may be already there.

There are various configurable mechanisms to implement this ownership mechanism. By default, TXT records are used, but other available options are AWS DynamoDB or AWS Service Discovery.

I will stick with the default TXT configuration because it doesn't require any other service running and it also allows me to quickly see, in my DNS provider, which records are created by ExternalDNS and which ones aren't.

## Installation

### ExternalDNS

ExternalDNS requires communication with a DNS provider, such as CloudFlare in this case, and to do so, it needs an API access token for authentication. The process for obtaining this token varies depending on the provider.

ExternalDNS [documentation](https://kubernetes-sigs.github.io/external-dns/v0.14.2/tutorials/cloudflare/) can guide you in the creation of the access token for your DNS provider of choice and list all the required permissions that should be granted when creating the token.

To make the provider access token available to ExternalDNS, we will put it in a Kubernetes secret that we will later reference. Let's proceed and create a secret with the following commands:

```sh
kubectl create namespace external-dns-system
kubectl create secret generic cloudflare-api-key --from-literal=apiKey=replace_this_with_your_secret -n external-dns-system
```

We will now proceed to install ExternalDNS using its Helm Chart.

1. Add the ExternalDNS repo to Helm

    ```sh
    helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
    helm repo update
    ```

1. Create a values file for ExternalDNS called `external-dns-values.yml` and paste the following markup:

    ```yml
    provider:
      name: cloudflare                # This is the name of your DNS provider
    env:
      - name: CF_API_TOKEN            # This is the environment variable where ExternalDNS expects to find the access token, which varies by provider so if you're not using CloudFlare, make sure you check the documentation
        valueFrom:
          secretKeyRef:
            name: cloudflare-api-key  # This should have the same name of the secret created above
            key: apiKey

    extraArgs:
      - --zone-id-filter=zone_id_here # This is useful if you have multiple zones (domains) in the same DNS provider, so ExternalDNS only monitors one
      - --traefik-disable-legacy      # Disable listeners on Resources under traefik.containo.us

    sources:                          # I'm using the Gateway, Service and Traefik as sources, the default is ingress and services only
      - gateway-httproute             # This is to analyze the Gateway HTTPRoute
      - traefik-proxy                 # This is to analyze Traefik's IngressRoute
      - service

    txtOwnerId: external-dns          # The value used to check records ownership with the standard ownership mechanism, TXT records

    policy: sync                      # How DNS records are synchronized between sources and providers; available values are `sync` & `upsert-only`. upsert-only doesn't delete records

    interval: 1m                      # This is the interval between DNS update, defaults to 1m

    # logLevel: debug                 # Uncomment this one to get a bit more logging, but don't get your hopes up, it's not nearly verbose enough...
    ```

1. Install the chart using the following command:

    ```sh
    helm upgrade --install external-dns external-dns/external-dns --values external-dns-values.yml -n external-dns-system
    ```

### Traefik & Gateway API (Optional)

This article is already lengthy, so I won’t delve into the details of the Gateway API here. That might be a topic for a future post. In brief, the Gateway API is a set of resources that will eventually replace the `Ingress` ones.

Since the Ingress resource has a limited configuration area, all the implementations (NGINX, Traefik, Kong, Envoy, etc) came up with different annotations to implement specific functionality. The Gateway resource is the response to this fragmentation, it allows you to specify a lot more behaviors in a vendor-agnostic way, so it’s a welcome addition.

If you're interested in exploring it, you'll need to install the CRDs and configure Traefik to support the Gateway API.

Before using the Gateway resource in Kubernetes you have to install the latest iteration of those CRDs (1.1.0 at the time of writing) using this command:

```sh
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/experimental-install.yaml
```

{{<note>}}
I won't configure TLS with the Gateway API because I haven't yet figured out how it works, I'd like to configure a different certificate for every site, but it seems to me that, at the moment, you either use a single TLS certificate with all the hostnames at the gateway controller level, or you have to use a `TLSRoute` that's still in the experimental channel at the time of writing (August 2024)
{{</note>}}

Traefik installation also needs to be updated to enable the kubernetesGateway provider, specify which namespaces are allowed to associate routes to the gateway and specify what to use as the Gateway's address via one of the supported ways.

Traefik's Gateway address can be configured in 3 different ways:

- Using a fixed IP address
- Using a hostname
- Using a service reference

I've opted to use the service reference, so I configured it to point to Traefik's own ingress controller service.

{{<tip>}}
If you want to know more about Traefik's Gateway address, make sure to check out the [documentation](https://doc.traefik.io/traefik/providers/kubernetes-gateway/#statusaddress)
{{</tip>}}

```yml
# I've added a comment to all the changed
# configuration values with an explanation
additionalArguments:
  # Sets logging level
  - "--log.level=DEBUG"
  # Opt out of telemetry
  - "--global.sendanonymoususage=false"
  # This is the name of the Traefik service, required for the Gateway API
  - "--providers.kubernetesgateway.statusaddress.service.name=traefik"
  # This is the namespace of the Traefik service, required for the Gateway API
  - "--providers.kubernetesgateway.statusaddress.service.namespace=traefik"

deployment:
  enabled: true
  replicas: 1

image:
  tag: 3.1.2 # Make sure to use a version greater than 3.1.1 otherwise the HTTPRoute approach will not work

ports:
  web:
    redirectTo:
      port: websecure
      priority: 10
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
  kubernetesIngress:
    enabled: false
  kubernetesGateway:
    enabled: true         # Enable the Gateway provider

rbac:
  enabled: true

service:
  enabled: true
  type: LoadBalancer

gateway:
  listeners:
    web:
      namespacePolicy: All  # Allow association for the web listener from all namespaces
    websecure: null         # Do not enable the HTTPS listener, due to the reason explained above
```

## Recipes

Now that we have installed and configured everything, we should be able to deploy a test website, wait a few minutes for ExternalDNS to do its job, and be able to resolve the service hostname with our cluster Traefik ingress controller service.

My test deployment is the classic NGINX website, exposed in different ways, I've verified the current setup works with the:

- Service type=LoadBalancer
- Traefik IngressRoute
- Gateway HTTPRoute

{{<tip>}}
Each recipe title tells you what resource is used to expose the test NGINX deployment and what record will be generated in the DNS provider.
{{</tip>}}

Here's the invariant part of the manifest, it's just a namespace and the NGINX deployment:

```yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-dns-site
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: demo-dns-site
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
# Paste the remainder of the configuration here below based on the
# chosen approach and apply using kubectl apply -f
```

Below you can find the manifest for each of the configurations I tested, copy and paste the configuration into the manifest here above:

### Service - A record

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo-dns-site
  annotations:
    external-dns.alpha.kubernetes.io/hostname: test.k8slab.app # This tells ExternalDNS which hostname to create
    external-dns.alpha.kubernetes.io/ttl: "300"                # This tells ExternalDNS what TTL to use on the DNS record
spec:
  type: LoadBalancer                                           # This tells kubernetes to expose the service to be externally accessible, MetalLB provides the IP address
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

The created DNS record looks like this:

```console
Type    Name        Content
A       test        192.168.2.210
TXT     a-test      "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=service/demo-dns-site/nginx-service"
TXT     test        "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=service/demo-dns-site/nginx-service"
```

### Service - CNAME record

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo-dns-site
  annotations:
    external-dns.alpha.kubernetes.io/target: k8slab.app        # This tells ExternalDNS which is the target of the hostname
    external-dns.alpha.kubernetes.io/hostname: test.k8slab.app # This tells ExternalDNS which hostname to create
    external-dns.alpha.kubernetes.io/ttl: "300"                # This tells ExternalDNS what TTL to use on the DNS record
spec:
  type: LoadBalancer                                           # This tells kubernetes to expose the service to be externally accessible, MetalLB provides the IP address
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

The created DNS record looks like this:

```console
Type    Name        Content
CNAME   test        k8slab.app
TXT     cname-test  "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=service/demo-dns-site/nginx-service"
TXT     test        "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=service/demo-dns-site/nginx-service"
```

### IngressRoute - A record

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo-dns-site
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-ingress-secure
  namespace: demo-dns-site
  annotations:
    external-dns.alpha.kubernetes.io/target: 192.168.2.210  # This tells ExternalDNS which is the target of the hostname
    external-dns.alpha.kubernetes.io/ttl: "300"             # This tells ExternalDNS what TTL to use on the DNS record
spec:
  entryPoints:
  - web
  - websecure
  routes:
  - match: Host(`test.k8slab.app`)                          # This tells ExternalDNS which hostname to create
    kind: Rule
    services:
    - name: nginx-service
      port: 80
```

The created DNS record looks like this:

```console
Type    Name        Content
A       test        192.168.2.210
TXT     a-test      "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=ingressroute/demo-dns-site/nginx-ingress-secure"
TXT     test        "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=ingressroute/demo-dns-site/nginx-ingress-secure"
```

### IngressRoute - CNAME record

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo-dns-site
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-ingress-secure
  namespace: demo-dns-site
  annotations:
    external-dns.alpha.kubernetes.io/target: k8slab.app     # This tells ExternalDNS which is the target of the hostname
    external-dns.alpha.kubernetes.io/ttl: "300"             # This tells ExternalDNS what TTL to use on the DNS record
spec:
  entryPoints:
  - web
  - websecure
  routes:
  - match: Host(`test.k8slab.app`)                          # This tells ExternalDNS which hostname to create
    kind: Rule
    services:
    - name: nginx-service
      port: 80
```

The created DNS record looks like this:

```console
Type    Name        Content
CNAME   test        k8slab.app
TXT     cname-test  "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=ingressroute/demo-dns-site/nginx-ingress-secure"
TXT     test        "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=ingressroute/demo-dns-site/nginx-ingress-secure"
```

### HTTPRoute - A record

{{<note>}}
This requires to install the Gateway API as documented [here](#traefik--gateway-api-optional)
{{</note>}}

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: demo-dns-site
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-service-http-route
  namespace: demo-dns-site
  annotations:
    external-dns.alpha.kubernetes.io/ttl: "300"             # This tells ExternalDNS what TTL to use on the DNS record
spec:
  parentRefs:   # This references the Traefik Gateway
    - name: traefik-gateway
      namespace: traefik

  hostnames:    # This specifies the hostname, same as Host(``) in IngressRoute
    - "test.k8slab.app"

  rules:
     - matches:
        - path:
            type: PathPrefix
            value: /

       backendRefs: # This specifies the target service
        - name: nginx-service
          namespace: demo-dns-site
          port: 80
```

The created DNS record looks like this:

```console
Type    Name        Content
A       test        192.168.2.210
TXT     cname-test  "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=httproute/demo-dns-site/nginx-service-http-route"
TXT     test        "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=httproute/demo-dns-site/nginx-service-http-route"
```

### HTTPRoute - CNAME record

{{<note>}}
This requires to install the Gateway API as documented [here](#traefik--gateway-api-optional)
{{</note>}}

To get a CNAME generated when using the gateway `HTTPRoute` resource, we need to add the target annotation on the gateway class itself, rather than on the `HTTPRoute` resource while our NGINX deployment can stay unchanged from the previous one.
Since the gateway class is created by Traefik, we need to change the values file used to install Traefik and install it again, you can see the updated value file below:

```yml
# I've added a comment to all the changed
# configuration values with an explanation
additionalArguments:
  # Sets logging level
  - "--log.level=DEBUG"
  # Opt out of telemetry
  - "--global.sendanonymoususage=false"
  # This is the name of the Traefik service
  - "--providers.kubernetesgateway.statusaddress.service.name=traefik"
  # This is the namespace of the Traefik service
  - "--providers.kubernetesgateway.statusaddress.service.namespace=traefik"

deployment:
  enabled: true
  replicas: 1

image:
  tag: 3.1.2 # Make sure to use a version greater than 3.1.1 otherwise the HTTPRoute approach will not work

ports:
  web:
    redirectTo:
      port: websecure
      priority: 10
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
  kubernetesIngress:
    enabled: false
  kubernetesGateway:
    enabled: true           # Enable the Gateway provider

rbac:
  enabled: true

service:
  enabled: true
  type: LoadBalancer

gateway:
  annotations:              # Added annotation to create a CNAME record
    external-dns.alpha.kubernetes.io/target: k8slab.app
  listeners:
    web:
      namespacePolicy: All  # Allow association for the web listener from all namespaces
    websecure: null         # Do not enable the HTTPS listener, due to the reason explained above
```

Now we can update the Traefik installation using the following command:

```sh
helm upgrade --install --namespace=traefik traefik traefik/traefik --values=traefikvalues.yml
```

The created DNS record looks like this:

```console
Type    Name        Content
A       test        192.168.2.210
TXT     cname-test  "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=httproute/demo-dns-site/nginx-service-http-route"
TXT     test        "heritage=external-dns,external-dns/owner=external-dns,external-dns/resource=httproute/demo-dns-site/nginx-service-http-route"
```

There you have it—now you know how to configure ExternalDNS with Traefik. You can easily choose between generating A records or CNAMEs using a `Service`, Traefik `IngressRoute`, or the Gateway `HTTPRoute`, depending on your network needs. In my case, I opted for A records since the IP of my ingress controller service remains constant.

## Troubleshooting

As I stated above, I ran into several issues while trying to set up ExternalDNS, so I decided to document here the issue and its solution:

### Failed to sync traefik.containo.us/v1alpha1, context deadline exceeded

This is the first issue I encountered as soon as I installed ExternalDNS using `traefik-proxy` as a source, this was the error message:

```sh
time="2024-07-28T15:19:44Z" level=fatal msg="failed to sync traefik.containo.us/v1alpha1, Resource=ingressrouteudps: context deadline exceeded"
```

This issue has been solved by disabling the legacy Traefik source using the argument `--traefik-disable-legacy` as explained [here](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/traefik-proxy.md#disabling-resource-listeners).

### No records are created for a Traefik IngressRoute

The second issue I found was the inability to generate any record at all when using Traefik `IngressRoute` as a source, the message found in the logs was the following:

```sh
time="2024-07-29T13:00:07Z" level=debug msg="Endpoints generated from service: demo-dns-site/nginx-service: []"
```

This was because my `IngressRoute` was missing the target annotation, such annotation specifies the IP or Hostname of the DNS target, if you omit this annotation while using an `IngressRoute`, nothing will be generated.

In my wishful thinking, I was hoping that ExternalDNS could resolve the Traefik service and use the external IP as a target but there's no relationship between the `IngressRoute` and the Traefik ingress controller service, so ExternalDNS explicitly requires you to specify a target annotation.

If the target is an IPv4, an A address is generated, if the target is an IPv6 an AAAA record is generated, otherwise the target is interpreted as a string and a CNAME is thus generated.

This annoyed me because it meant that the configuration had to be repeated on every `IngressRoute`, making it more difficult to change, especially in an environment that's mostly a test cluster. I decided then to look into the new Kubernetes networking resource, the Gateway API.

### Gateway traefik/traefik-gateway has not accepted HTTPRoute demo-dns-site/nginx-service-http-route

The Gateway resources, `HTTPRoute` and `GRPCRoute` resources have to specify the parent Gateway controller instance. A Gateway controller is in turn assigned an address. This seems perfect to avoid duplication I was facing with the `IngressRoute` option so I decided to replace the Traefik `IngressRoute` with a Gateway `HTTPRoute`, and immediately ran into the following error:

```sh
level=debug msg="Gateway traefik/traefik-gateway has not accepted HTTPRoute demo-dns-site/nginx-service-http-route"
```

The issue this time was related to Traefik, to resolve it I had to update the Traefik installation by adding the following to the values file:

```yml
gateway:
  listeners:
    web:
      namespacePolicy: All  # Allow resource from all namespaces to be accepted by the gateway
    websecure: null         # Do not install the websecure listener
```

### No records are created for an HTTPRoute

As soon as the route acceptance issue was fixed after re-deploying Traefik, the new error was the good old: Hey I don't know what DNS records to create for this `HTTPRoute` 🤷‍♂️.

To troubleshoot this issue I went down a rabbit hole, I decided to clone both ExternalDNS and Traefik code bases and added some logging because, truth be told, ExternalDNS with debug logging enabled is still not logging enough information to easily troubleshoot failures.

```sh
time="2024-08-07T06:07:39Z" level=debug msg="Endpoints generated from HTTPRoute demo-dns-site/nginx-service-http-route: []"
```

While troubleshooting this issue I figured that my Gateway didn't get an address. That turned out to be related to some missing Traefik **static configuration**

As shown earlier, Traefik's Gateway Address can be configured in 3 different ways:

- Using a fixed IP address
- Using a hostname
- Using a service reference

But none of them worked, after a deeper investigation, it turned out to be a recently introduced issue in Traefik itself that I [reported](https://github.com/traefik/traefik/pull/10940#discussion_r1702623183) and got promptly fixed [the day after](https://github.com/traefik/traefik/pull/10972). Good job Traefik Labs! The fix has been released in versions 3.1.2 and 2.11.7.

## References

- [ExternalDNS documentation](https://kubernetes-sigs.github.io/external-dns/v0.14.2/)
- [Traefik configuration overview](https://doc.traefik.io/traefik/getting-started/configuration-overview/)
- [Traefik Gateway provider configuration](https://doc.traefik.io/traefik/providers/kubernetes-gateway/#statusaddress)

## Conclusion

As you can see this was not as straightforward as expected, but in the end I managed to configure ExternalDNS and have it sync my DNS records correctly.

With cert-manager and ExternalDNS working together, we can fully automate DNS management. This setup allows us to define DNS configurations and TLS certificates programmatically through Kubernetes manifests, making it incredibly convenient and efficient.

One more step in the homelab journey automation!

Till the next time.
