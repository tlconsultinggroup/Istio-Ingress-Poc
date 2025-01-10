## Steps
* Create a namespace public-ingress for Ingress Controller where all ingress controller-related resources will be created.
* Install cert-manager for SSL certificates in public-ingress namespace using Helm.
* Create a CA cluster issuer for issuing certificates.
* Create deployments and services.
* Setup A Record of domain
* Create an ingress route to configure the rules that route traffic to service.
* Verify the automatic created certificate.
* Test the applications using Custom Domain.

1. Create an ingress controller

```bash
# Create a namespace for ingress resources
kubectl create namespace public-ingress

# Add the Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Use Helm to deploy an NGINX ingress controller
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace public-ingress \
  --set controller.config.http2=true \
  --set controller.config.http2-push="on" \
  --set controller.config.http2-push-preload="on" \
  --set controller.ingressClassByName=true \
  --set controller.ingressClassResource.controllerValue=k8s.io/ingress-nginx \
  --set controller.ingressClassResource.enabled=true \
  --set controller.ingressClassResource.name=public \
  --set controller.service.externalTrafficPolicy=Local \
  --set controller.setAsDefaultIngress=true
```
2. Install cert-manager

```bash
# Label the cert-manager namespace to disable resource validation
kubectl label namespace public-ingress cert-manager.io/disable-validation=true
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
# Update your local Helm chart repository cache
helm repo update
# Install CRDs with kubectl
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace public-ingress \
  --version v1.11.0
```
3. Create a CA cluster issuer

```bash
# Cluster Issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: #Use your mail id
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - http01:
          ingress:
            class: public

```

4. Run Demo application

We have used the same Azure OpenAI application.

```bash
kubectl apply -f https://raw.githubusercontent.com/Azure-Samples/aks-store-demo/main/aks-store-all-in-one.yaml
``` 


You can either deploy the sample application provided for testing, or trigger sidecar injection for existing workloads.

If you have existing applications to be added to the mesh, ensure their namespaces are labeled as in the previous step, and then restart their deployments to trigger sidecar injection:
```bash
kubectl rollout restart -n <namespace> <deployment name>
```
Verify that sidecar injection succeeded by ensuring all containers are ready and looking for the istio-proxy container in the kubectl describe output, for example:

```bash
kubectl describe pod -n namespace <pod name>
```
5. Create an A record

  * A record of custom domain will be the Public IP of Ingress Controller. Use this command to get external ip of nginx controller and create A-Record for you domain.

```bash
kubectl get svc -n public-ingress
```
6. Create an ingress route
The ingress resource configures the rules that route traffic to the application.

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/use-regex: "true"
    
spec:
  ingressClassName: public
  tls:
  - hosts:
    - cert-test.online #Use your domain
    secretName: tls-secret
  rules:
  - host: cert-test.online #Use your domain
    http:
      paths:
      - path: "/"
        pathType: Prefix
        backend:
          service:
            name: store-admin
            port:
              number: 8081
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: store-admin
            port:
              number: 8081
      - path: /product-service
        pathType: Prefix
        backend:
          service:
            name: product-service
            port:
              number: 3002
      - path: /makeline-service
        pathType: Prefix
        backend:
          service:
            name: makeline-service
            port:
              number: 3001
```

7. Verify certificate
```bash
kubectl get certificate  --namespace public-ingress
```

8. Test the ingress configuration
https://cert-test.online/
Open a web browser to the FQDN of your Kubernetes ingress controller, such as https://cert-test.online/

