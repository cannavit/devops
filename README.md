# Continuos Integration Architecture (K8s)

This document contains all the necessary steps to develop a continuous integration (CI) architecture based on Kubernetes, Terraform, Nginx, RDS, and a load balancing strategy using CANARY. This type of architecture is recommended for high availability services with continuous deployment of new versions.


![k8s](docs/k8s_canary_postgres_redis_rds/k8s.png ':size=100%')


Kubernetes is a fantastic tool for this type of case.

> Maintainer

This documentation is supported by [CannaviT](https://github.com/cannavit)


<a href="https://www.buymeacoffee.com/cannavit" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="41" width="174"></a>
###  Table of contents

1. [Kubernetes TLS Cerfificates](#tls-certificates)
2. [Cannary Strategy with Flagger](#canary-flagger)



## Configuration of NGINX Ingress (Only with Init Cluster)

> Helm  Chart:

[ingress-nginx](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx) Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer

Add repository

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

Install chart

    helm install my-ingress-nginx ingress-nginx/ingress-nginx --version 4.1.1

Copy the output inside of the file

    k8s/staging/incloodo-backend/02-incloodo-backend-secrets.yaml


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

    kubectl  describe  certificates/incloodo-staging-tls -n incloodo-staging



## Connect Service DB to RDS of AWS

For connect one service to RDS of the aws you need create the next service configuration:

    apiVersion: v1
    kind: Service
    metadata:
      name: postgresql-rds
      namespace: "example-staging"
      labels:
        app.kubernetes.io/name: postgresql
        app: example-backend
        run: example
    spec:
      externalName: XXXXXXXXXXX.eu-south-1.rds.amazonaws.com
      selector:
        app: postgresql
      type: ExternalName
    status:
      loadBalancer: {}



## Cannary Strategy with Flagger <a name="tls-certificates"></a>

### Requirements

You must meet the following requirements: 

- [Helm](https://helm.sh/docs/intro/install/#helm)
- [kubectl client](https://kubernetes.io/docs/tasks/tools/)
- [Ingress Services](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [HPA](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)

What is [Flagger](https://flagger.app/) Flagger's application analysis can be extended with metric queries targeting Prometheus, Datadog, CloudWatch, New Relic, Graphite, Dynatrace, InfluxDB and Google Cloud Monitoring (Stackdriver).

Flagger can be configured to send notifications (opens new window)to Slack, Microsoft Teams, Discord and Rocket. It will post messages when a deployment has been initialised, when a new revision has been detected and if the canary analysis failed or succeeded.

![flager-example](https://flagger.app/flagger-gitops.png ':size=100%')

### NGINX Canary Deployments

This [guide](https://docs.flagger.app/tutorials/nginx-progressive-delivery) shows you how to use the NGINX ingress controller and Flagger to automate canary deployments and A/B testing.

#### Install Flagger

1.Update NGINX

    helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
    
upgrade

    helm upgrade -i my-ingress-nginx ingress-nginx/ingress-nginx \
    --namespace default \
    --set controller.metrics.enabled=true \
    --set controller.podAnnotations."prometheus\.io/scrape"=true \
    --set controller.podAnnotations."prometheus\.io/port"=10254

1.Get the name of the flagger with prometheus

    helm repo add flagger https://flagger.app

    helm upgrade -i flagger flagger/flagger \
      --namespace default \
      --set prometheus.install=true \
      --set meshProvider=nginx




2.Add labels in your current ingress

  kubectl label ingress/ingress-example-backend -n $NAMESPACE  canary=example-flugger-prod


Create a canary custom resource (replace app.example.com with your own domain):

Add of parameters of your cluster

    apiVersion: flagger.app/v1beta1
    kind: Canary
    metadata:
      name: podinfo
      namespace: test
    spec:
      provider: nginx
      # deployment reference
      targetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: podinfo    <<<<<<HERE with: kubectl get deployments -n $NAMESPACE >>>>>>
      # ingress reference
      ingressRef:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: podinfo  <<<<<<HERE with: kubectl get ingress -n $NAMESPACE >>>>>>
      # HPA reference (optional)
      autoscalerRef:
        apiVersion: autoscaling/v2beta2
        kind: HorizontalPodAutoscaler
        name: podinfo   <<<<<<HERE with: kubectl get hpa -n $NAMESPACE >>>>>> 
      # the maximum time in seconds for the canary deployment
      # to make progress before it is rollback (default 600s)
      progressDeadlineSeconds: 60
      service:
        # ClusterIP port number
        port: 80
        # container port number or name
        targetPort: 9898
      analysis:
        # schedule interval (default 60s)
        interval: 10s
        # max number of failed metric checks before rollback
        threshold: 10
        # max traffic percentage routed to canary
        # percentage (0-100)
        maxWeight: 50
        # canary increment step
        # percentage (0-100)
        stepWeight: 5
        # NGINX Prometheus checks
        metrics:
        - name: request-success-rate
          # minimum req success rate (non 5xx responses)
          # percentage (0-100)
          thresholdRange:
            min: 99
          interval: 1m
        # testing (optional)
        webhooks:
          - name: acceptance-test
            type: pre-rollout
            url: http://flagger-loadtester.test/
            timeout: 30s
            metadata:
              type: bash
              cmd: "curl -sd 'test' http://podinfo-canary/token | grep token"
          - name: load-test
            url: http://flagger-loadtester.test/
            timeout: 5s
            metadata:
              cmd: "hey -z 1m -q 10 -c 2 http://app.example.com/"


                               <!-- set image deployment/example-backend example-backend=${{ env.RELEASE_IMAGE }} --record -n $KUBE_NAMESPACE   

Deploy example

    kubectl -n example-production set image deployment/example-backend example-backend=ghcr.io/example/backend:production_1.25

Get information in canary

    kubectl describe canary/example-backend -n $NAMESPACE 

Get all canaries 

    kubectl get canaries --all-namespaces





ingress-example-backend
ingress-example-backend-canary -->


kubectl debug example-staging -it --image=ghcr.io/example/backend --share-processes 



## Utils 
### Connect with DB from inside of container:


    apk add postgresql-client  
    pqsl -h postgresql -U postgres 
    psql -h postgresql  -U postgres incloodo_stagin   
    psql -h postgresql-2  -U postgres incloodo_production
    psql -h postgresql-rds  -U postgres incloodo_production




    kubectl -n incloodo-production set image deployment/incloodo-backend incloodo-backend=ghcr.io/incloodo/backend:production_latest

    kubectl -n incloodo-production set image deployment/incloodo-backend incloodo-backend=ghcr.io/incloodo/backend:production_1.40
    kubectl -n incloodo-production set image deployment/incloodo-backend incloodo-backend=ghcr.io/incloodo/backend:production_1.44



    kubectl describe canary/incloodo-backend -n incloodo-production 

    kubectl get canaries --all-namespaces