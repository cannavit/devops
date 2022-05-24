# Welcome to K8s Configuration


Here is all the documentation to implement the DevOps culture as a PRO. 


> Maintainer

This documentation is supported by [CannaviT](https://github.com/cannavit)


<a href="https://www.buymeacoffee.com/cannavit" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="41" width="174"></a>
###  Table of contents

1. [Kubernetes TLS Cerfificates](#tls-certificates)


## Configuration of TLS certificate <a name="tls-certificates"></a>


### Requirements

You must meet the following requirements: 

- [Helm](https://helm.sh/docs/intro/install/#helm)
- [kubectl client](https://kubernetes.io/docs/tasks/tools/)
- [Ingress Services](https://kubernetes.io/docs/concepts/services-networking/ingress/)


For create TLS secure certificates i go to use, [cert-manager](https://cert-manager.io/) is a powerful and extensible X.509 certificate controller for Kubernetes and OpenShift workloads. It will obtain certificates from a variety of Issuers, both popular public Issuers as well as private Issuers, and ensure the certificates are valid and up-to-date, and will attempt to renew certificates at a configured time before expiry.

![logo-example](https://cert-manager.io/images/cert-manager-graphic.svg ':size=100%')

#### Steps

1. Add the Jetstack Helm repository

        helm repo add jetstack https://charts.jetstack.io

2. Update your local Helm chart repository cache:

        helm repo update

3. Install

        kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.0/cert-manager.crds.yaml

4. Install cert-manager

        helm install \
            cert-manager jetstack/cert-manager \
            --namespace cert-manager \
            --create-namespace \
            --version v1.8.0

5. Creating a Basic [ACME](https://cert-manager.io/docs/configuration/acme/#creating-a-basic-acme-issuer) Issue

[Issuers](https://cert-manager.io/docs/concepts/issuer/), and ClusterIssuers, are Kubernetes resources that represent certificate authorities (CAs) that are able to generate signed certificates by honoring certificate signing requests. All cert-manager certificates require a referenced issuer that is in a ready condition to attempt to honor the request.


Copy this in one file and apply the ClusterInssuer

    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        # You must replace this email address with your own.
        # Let's Encrypt will use this to contact you about expiring
        # certificates, and issues related to your account.
        email: cecilio.cannav@gmail.com
        server: https://acme-v02.api.letsencrypt.org/directory
        privateKeySecretRef:
          # Secret resource that will be used to store the account's private key.
          name: letsencrypt-prod
        # Add a single challenge solver, HTTP01 using nginx
        solvers:
        - http01:
            ingress:
              class: nginx

6. Edit your ingress configuration and add this:

Add this lines

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: ingress-example-backend
      namespace: example-production
      annotations:
        kubernetes.io/ingress.class: "nginx"           <<< ADD THIS LINE >>>
        cert-manager.io/cluster-issuer: "letsencrypt-prod"  <<< ADD THIS LINE >>>
    spec:
      ingressClassName: nginx
      rules:
        - host: server.example.com
          http:
            paths:
              - pathType: Prefix
                backend:
                  service:
                    name: example-backend
                    port:
                      number: 8000
                path: /
      tls:                        <<< ADD THIS LINE >>>
        - hosts:                   <<< ADD THIS LINE >>>
          - server.example.com   <<< ADD THIS LINE >>>
          secretName: example-production-tls <<< ADD THIS LINE >>>


#### Verify certificate 

    kubectl get certificates $NAMESPACE

expected response:

    NAMESPACE             NAME                      READY   SECRET                    AGE
    example-production   example-production-tls     True    example-production-tls    90s

describe certificate:

    kubectl  describe  certificates/example-production-tls -n $NAMESPACE

