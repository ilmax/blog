---
title: "Install K3s on a Raspberry PI - TLS certificates"
date: 2024-07-16T20:45:15+02:00
draft: true
series: ["K3s on Raspberry PI"]
series_order: 3
tags: ["Kubernetes", "homelab", "rpi", "tutorial"]
---

If you are following along in the series, so far we have configured the master node, added a few worker nodes to the cluster and that's pretty much about it. Now it's time to implement a fun part, the SSL certificate management for our internally exposed web applications, so without further ado, let's dive right into it.

## cert-manager, Let's Encrypt & Cloudflare

Cert-manager is a certificate controller for Kubernetes, created by [Jetstack](https://venafi.com/jetstack-consult/) and donated to CNCF. Amongst its key features, we are specifically interested in:

- Automated issuance and renewal of certificates to secure Ingress with TLS
- Fully integrated Issuers from recognized public and private Certificate Authorities

We will install cert-manager and configure it to request a certificate from Let's Encrypt CA.

I won't go into great details on how the certificate issuing process works but, given that our services are not exposed to the internet but only internally in our network, we need to pick a Let'sEncrypt challenge that supports this scenario.

### Let's Encrypt challenges

As the [documentation](https://letsencrypt.org/docs/challenge-types/) puts it

"*When you get a certificate from Let’s Encrypt, our servers validate that you control the domain names in that certificate using challenges*"

Let's Encrypt supports several challenge types:

- HTTP-01
- DNS-01
- TLS-ALPN-01

Out of those 3, HTTP-01 and TLS-ALPN-01 require your server to be exposed to the internet, so the only suitable challenge we can pick is DNS-01.

### The DNS-01 challenge

The DNS-01 challenge steps are represented in the following sequence diagram:

{{<figure src="dns01.svg" alt="Let's Encrypt DNS01 challenge" caption="*The diagram DNS-01 challenge*" nozoom=true >}}

{{<alert icon="lightbulb">}}
If you want to read more about how the ACME protocol works, you can read the internet standard [RFC 8555](https://datatracker.ietf.org/doc/html/rfc8555)
{{</alert>}}

### DNS Registrar

I'm using Cloudflare as my registrar because I love their services, all the functionality I need from Cloudflare is supported by their API and there are a lot of integrations and it works natively with cert-manager.

The list of supported providers for the DNS-01 challenge can be found [here](https://cert-manager.io/docs/configuration/acme/dns01/#supported-dns01-providers).

## Install and configure cert-manager

We're using Helm to install cert-manager, so we need to add the Helm repository and then install the chart.

```sh
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.15.1 \
  --set crds.enabled=true
```

### Generate Cloudflare API token

Now that we installed cert-manager, we have to configure it to consume the Cloudflare API, so the first thing required is to create a Cloudflare API token with the required permissions as per the documentation here:

- Permissions:

  - `Zone - DNS - Edit`
  - `Zone - Zone - Read`

- Zone Resources

  - `Include - All Zones`

Now we need to create a Kubernetes secret with the token generated earlier:

```sh
kubectl create secret generic cloudflare-api-key-secret \
  -n cert-manager --from-literal=api-key=[YOUR_CLOUDFLARE_API_KEY]
```

## Configure the staging ClusterIssuer

When we install cert-manager, it creates some custom resource definitions (CRDs) to configure various aspects of the certificate request process, two of which are the `Issuer` or `ClusterIssuer`. Those resources represent certificate authorities (CAs) so this is the resource that will issue our cluster the certificate.

Both resources are pretty much identical, with the only difference being that one, the `Issuer` is namespace scoped, while the other `ClusterIssuer` is global, so if you use the issuer and need a certificate for services in multiple namespaces, you need to deploy multiple `Issuer`s while with the `ClusterIssuer` one resource suffice for the whole cluster. I've picked the `ClusterIssuer` one.

On The `ClusterIssuer` we have to configure Let's Encrypt DNS-01 challenge and point the resource to the secret created earlier used to authenticate towards the Cloudflare API.

Due to the strict rate limiting applied by the Let's Encrypt production services, it's a good idea to start using their staging APIs and then, when everything works as expected, create the production `ClusterIssuer` and use that one.

 let's see here below what the configuration looks like:

1. Create a file file called `clusterissuer-staging.yml` and paste the following manifest into it:

   ```yml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-dns01-staging-issuer
   spec:
     acme:
       server: https://acme-staging-v02.api.letsencrypt.org/directory  # Staging API
       email: mail@mail.com                     # your email address for updates
       privateKeySecretRef:
         name: letsencrypt-dns01-staging-private-key # Name of a secret used to store the ACME account private key
       solvers:
       - dns01:
           cloudflare:
             email: mail@mail.com               # your cloudflare account email address
             apiTokenSecretRef:
               name: cloudflare-api-key-secret  # Matches the name of the secret created earlier
               key: api-key                     # Matches the key of the secret created earlier
   ```

1. Create the staging `ClusterIssuer` in the cluster

    ```sh
    kubectl apply -f clusterissuer-staging.yml
    ```

{{<alert>}}
If you use a different namespace than `cert-issuer`, you may need to configure the Cluster Issuer Namespace to specify cert-manager in which namespace to look for the Cloudflare secret. Make sure to read the documentation [here](https://cert-manager.io/docs/configuration/#cluster-resource-namespace)
{{</alert>}}

### Verify staging ClusterIssuer installation

To make sure everything is configured correctly, we will create a certificate issue request since we're using the staging API of Let's Encrypt, this certificate won't be used for TLS, only to make sure everything is configured correctly.

Before doing so, let's create an A record in Cloudflare that points to the Traefik service created in the first article of the series, to get the IP address let's use the following command:

```sh
kubectl get svc -n traefik
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                    AGE
traefik   LoadBalancer   10.43.63.113   192.168.2.210   80:31468/TCP,443:31486/TCP,443:31486/UDP   13d
```

Here we are interested in the external IP, so we will create an A record that points to that IP. Please note that we are creating an A record in our DNS that points to a **private IP address** because we want to expose our services only to the internal home network and not make them available over the internet.

For this example, I've created a temporary A record k3s in the `maxdon.tech` domain I own that points to 192.168.2.210. After the record has been created, it's time to test the certificate-issuing process.

1. Create a test namespace

    ```sh
    kubectl create namespace test-cert
    ```

1. Create a certificate issue request by pasting the following code in a file called `test-certificate-staging.yml`

   ```yml
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
     name: test-certificate
     namespace: test-cert
   spec:
     secretName: test-example-tls             # This is the name of the secret that will hold the TLS certificate
     issuerRef:
       name: letsencrypt-dns01-staging-issuer # This should be the name of the staging CLusterIssuer
       kind: ClusterIssuer
     dnsNames:
     - k3s.maxdon.tech                        # This should be the same name of the A record created in Cloudflare earlier
   ```
   
  {{<alert cardColor="#e63946" iconColor="#1d3557">}}
  Make sure you replace the domain maxdon.tech with the domain you own.
  {{</alert>}}

1. Create the test certificate in the cluster

    ```sh
    kubectl apply -f test-certificate-staging.yml
    ```

1. Verify that the certificate has been issued with `kubectl get certificates` and the output should be the following:

```sh
kubectl get certificate -n test-cert
NAME               READY   SECRET             AGE
test-certificate   True   test-example-tls    75s
```

{{<alert icon="info-solid">}}
Please note that this step can take up to a couple of minutes when using the DNS-01 challenge!
{{</alert>}}

If the process went well, we now have a secret that contains our certificate, the name of the secret is defined when we create the certificate resource, we can then inspect the secret using this command:

```sh
kubect describe -n test-cert secrets test-example-tls
Name:         test-example-tls
Namespace:    test-cert
Labels:       controller.cert-manager.io/fao=true
Annotations:  cert-manager.io/alt-names: k3s.maxdon.tech
              cert-manager.io/certificate-name: test-certificate
              cert-manager.io/common-name: k3s.maxdon.tech
              cert-manager.io/ip-sans:
              cert-manager.io/issuer-group:
              cert-manager.io/issuer-kind: ClusterIssuer
              cert-manager.io/issuer-name: letsencrypt-dns01-staging-issuer
              cert-manager.io/uri-sans:

Type:  kubernetes.io/tls

Data
====
tls.crt:  3733 bytes
tls.key:  1679 bytes
```

### Troubleshooting

If the certificate-issuing process fails, here are a few things to look out for:

1. Make sure that the `ClusterIssuer` is **ready**

    ```sh
    kubectl get clusterissuer
    NAME                               READY   AGE
    letsencrypt-dns01-staging-issuer   True    2m7s
    ```

    If ready is false, try to look at the events of the `ClusterIssuer` using the following command:

    ```sh
    kubectl describe clusterissuer letsencrypt-dns01-staging-issuer
    ```

    I had the following error:

    ```yml
    Status:
    Acme:
    Conditions:
        Last Transition Time:  2024-07-17T12:57:39Z
        Message:               Failed to register ACME account: Get "https://acme-staging-v02.api.letsencrypt.org/directory": dial tcp: lookup acme-staging-v02.api.letsencrypt.org on 10.43.0.10:53: server misbehaving
        Observed Generation:   1
        Reason:                ErrRegisterACMEAccount
        Status:                False
        Type:                  Ready
    ```

    This was due to an incorrect configuration of my DNS so the cluster couldn't correctly resolve hostnames, fixing the DNS issue and deleting and re-installing the `ClusterIssuer` fixed the issue.

    For additional troubleshooting tips, refer to the cert-manager documentation on the subject [here](https://cert-manager.io/docs/troubleshooting/acme/#1-troubleshooting-clusterissuers).

1. Check the issuing process

    With cert-manager, whenever we create a `Certificate` resource, those resources are linked and you can "walk" the link using `kubectl describe {resourcetype} {resourcename}` using the order as a starting point. What we are interested in here is the `Events:` section of the output which will help us figure out what's wrong. Another way to achieve the same result, is to query for all the events in the namespace we care about:

    ```sh
    kubectl get events -n test-cert
    LAST SEEN   TYPE      REASON               OBJECT                                                     MESSAGE
    32m         Normal    Started              challenge/test-certificate-wrong-1-4090149863-2764006135   Challenge scheduled for processing
    11m         Warning   PresentError         challenge/test-certificate-wrong-1-4090149863-2764006135   Error presenting challenge: Found no Zones for domain _acme-challenge.k3s.example.tech. (neither in the sub-domain nor in the SLD) please make sure your domain-entries in the config are correct and the API key is correctly setup with Zone.read rights.
    32m         Normal    Created              order/test-certificate-wrong-1-4090149863                  Created Challenge resource "test-certificate-wrong-1-4090149863-2764006135" for domain "k3s.example.tech"
    32m         Normal    WaitingForApproval   certificaterequest/test-certificate-wrong-1                Not signing CertificateRequest until it is Approved
    32m         Normal    WaitingForApproval   certificaterequest/test-certificate-wrong-1                Not signing CertificateRequest until it is Approved
    32m         Normal    WaitingForApproval   certificaterequest/test-certificate-wrong-1                Not signing CertificateRequest until it is Approved
    32m         Normal    WaitingForApproval   certificaterequest/test-certificate-wrong-1                Not signing CertificateRequest until it is Approved
    32m         Normal    WaitingForApproval   certificaterequest/test-certificate-wrong-1                Not signing CertificateRequest until it is Approved
    32m         Normal    cert-manager.io      certificaterequest/test-certificate-wrong-1                Certificate request has been approved by cert-manager.io
    32m         Normal    OrderCreated         certificaterequest/test-certificate-wrong-1                Created Order resource test-cert/test-certificate-wrong-1-4090149863
    32m         Normal    Issuing              certificate/test-certificate-wrong                         Issuing certificate as Secret does not exist
    32m         Normal    Generated            certificate/test-certificate-wrong                         Stored new private key in temporary Secret resource "test-certificate-wrong-rvtwc"
    32m         Normal    Requested            certificate/test-certificate-wrong                         Created new CertificateRequest resource "test-certificate-wrong-1"
    ```

    From here you can see that I requested a certificate for a domain I don't own (example.com)

### Cleanup the test certificate

If you successfully managed to get a certificate using the staging `ClusterIssuer`, now it's time to clean up the test certificate and the relative secret, the quickest way to do so is to delete the whole namespace:

```sh
kubectl delete ns test-cert
```

## Configure the production ClusterIssuer

Now, that we correctly obtained a staging certificate, it's time to configure cert-manager to use the production API of Let's Encrypt, in order to do so, we need to change the server URI and give it a different name, everything else stays exactly the same as the staging `ClusterIssuer`, see the updated manifest here below:

1. Create a file file called `clusterissuer-production.yml` and paste the following manifest into it:

  ```yml
  apiVersion: cert-manager.io/v1
  kind: ClusterIssuer
  metadata:
    name: letsencrypt-dns01-production-issuer
  spec:
    acme:
      server: https://acme-v02.api.letsencrypt.org/directory    # Production API
      email: mail@mail.com                             # your email address for updates
      privateKeySecretRef:
        name: letsencrypt-dns01-production-private-key # Name of a secret used to store the ACME account private key
      solvers:
      - dns01:
          cloudflare:
            email: mail@mail.com                       # your cloudflare account email address
            apiTokenSecretRef:
              name: cloudflare-api-key-secret
              key: api-key
  ```

1. Create the staging `ClusterIssuer` in the cluster

  ```sh
  kubectl apply -f clusterissuer-production.yml
  ```

At this point, after making sure that the production `ClusterIssuer` is ready as we did previously using:

```sh
kubectl get clusterissuer
```

### Verify production ClusterIssuer installation

We can deploy a demo site using nginx and request a valid certificate from Let's Encrypt, copy and paste the following markup in a faile called `demosite.yml`:

{{<alert cardColor="#e63946" iconColor="#1d3557">}}
Make sure you replace the domain maxdon.tech with the domain you own.
{{</alert>}}

```yml
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-site
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-web
  namespace: demo-site
  labels:
    app: nginx-web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-web
  template:
    metadata:
      labels:
        app: nginx-web
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-web-service
  namespace: demo-site
  labels:
    app: nginx-web
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-web
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: k3s-tls-certificate
  namespace: demo-site
spec:
  secretName: k3s-maxdon-tech-tls             # Specify the name of the generated TLS certificate secret
  issuerRef:
    name: letsencrypt-dns01-production-issuer # Use the production ClusterIssuer
    kind: ClusterIssuer
  dnsNames:
  - k3s.maxdon.tech
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nginx-web-ingress
  namespace: demo-site
spec:
  entryPoints:
  - websecure                                 # Use websecure bind on 443 that's an entry point defined by Traefik, alongside a web one that it's bind on port 80
  routes:
  - match: Host(`k3s.maxdon.tech`)
    kind: Rule
    services:
    - name: nginx-web-service
      port: 80
  tls:
    secretName: k3s-maxdon-tech-tls         # Match the name of the secret that contains the certificate
```

Now let's create all the resources in the cluster using our friend kubeclt as follows:

```sh
kubectl apply -f demosite.yml
```

After few minutes, you should be able to see a certificate in the demo-site namespace uwing the following command:

```sh
kubectl get certificate -n demo-site
NAME                  READY   SECRET                AGE
k3s-tls-certificate   True    k3s-maxdon-tech-tls   92s
```

At this point, we should be able to navigate in our browser to the website and verify that we have a valid TLS certificate, that has been issued by Let's Encrypt, we can now delete the demosite resources using the following command:

```sh
kubectl delete -f demosite.yml
```

## Conclusion

This was quite a lengthy post, but cert-manager makes it really easy for us to configure and manage automated certificate management. cert-manager doesn't support all the domain registrar so if you still have to buy a domain, make sure that you're buying it from one of the supported registrar.

Just to recap, in order to expose a service with it's relative TLS certificate we need to:

1. Create a certificate resource to instruct cert-manager to request a certificate for us
1. Instruct Traefik which certificate to use, specifying the same secret name used in the certificate resource

At this point we implemented all the features from the first post:

- Multinode k8s Cluster {{<iconc "check" "green">}}
- Do not expose any internal service to the internet {{<iconc "check" "green">}}
- Resolve internal services deployed on the cluster via domain name {{<iconc "check" "green">}}
- Have free SSL certificates and automated certificate renewal {{<iconc "check" "green">}}

Pretty easy no?

In the next article we will look at automating the DNS record generation in the DNS registrar, so stay tuned for th next one!
