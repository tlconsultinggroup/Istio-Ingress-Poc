# Cert-Manager Let's Encrypt Integration with Istio-based service mesh add-on in Azure AKS 

Please follow up below steps to integrate Istio-based service mesh add-on for AKS with cert-manager and obtain let's encrypt certificates for setting up secure ingress gateways.

## Before you begin

* [Install](https://learn.microsoft.com/en-us/azure/aks/istio-deploy-addon#install-istio-add-on) Istio-based service mesh add-on on your cluster.

```bash
az aks mesh enable -g <rg-name> -n <cluster-name>
```
* [Enable external ingressgateway](https://learn.microsoft.com/en-us/azure/aks/istio-deploy-ingress#enable-external-ingress-gateway)

```bash
az aks mesh enable-ingress-gateway -g <rg-name> -n <cluster-name> --ingress-gateway-type external
```
* [Enable sidecar injection](https://learn.microsoft.com/en-us/azure/aks/istio-deploy-addon#enable-sidecar-injection) on the default namespace

```bash
kubectl label namespace default istio.io/rev=$revision
```
## Steps

### 1. Setup DNS record

[Set up a DNS record](https://learn.microsoft.com/en-us/azure/dns/dns-operations-recordsets-portal)
for the EXTERNAL-IP address of the external ingressgateway service with Azure DNS.

Run the following command to retrieve the external IP address of the ingress gateway:

```bash
kubectl get svc -n aks-istio-ingress
```
```bash
NAME                                TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)                                      AGE
aks-istio-ingressgateway-external   LoadBalancer   10.0.88.165     4.195.18.233    15021:30786/TCP,80:30626/TCP,443:30236/TCP   4h44m
```
Verify/wait until dig +short A cert-test.online returns the configured IP address that is 4.195.18.233 in this example

### 2. Install demo app

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.21/samples/bookinfo/platform/kube/bookinfo.yaml
```
Confirm several deployments and services are created on your cluster and ensure each pod has 2/2 containers in the Ready state

```bash
NAME                              READY   STATUS    RESTARTS   AGE
productpage-v1-5bb9985d4d-2f58v   2/2     Running   0          3h28m
ratings-v1-6484d64bbc-khjkk       2/2     Running   0          3h28m
reviews-v1-598f9b58fc-kxwwd       2/2     Running   0          3h28m
reviews-v2-5979c6fc9c-k8qnj       2/2     Running   0          3h28m
reviews-v3-7bbb5b9cf7-7zw8n       2/2     Running   0          3h28m

```
### 3. Configure ingress gateway and virtual service

Before deploying the virtualservice and gateway resources, make sure to update the host name to match the previously created Azure DNS record name.

```bash
kubectl apply -f istio-gateway.yaml
kubectl apply -f istio-virtualservice.yaml
```

### Note

In the gateway definition, credentialName must match the secretName in the certificate resource which will be created later in the example and selector must refer to the external ingress gateway by its label, in which the key of the label is istio and the value is aks-istio-ingressgateway-external.

### Validate http request to productpage service

Send an HTTP request to access the productpage service

```bash
curl -sS http://cert-test.online/productpage | grep -o "<title>.*</title>"
```
this should print ``` <title>Simple Bookstore App</title> ```

### 4. Create the shared configmap
Create a ConfigMap with the name istio-shared-configmap-<asm-revision> in the aks-istio-system namespace to set ingressService and ingressSelector. For example, if AKS cluster is running asm-1-22 revision of mesh, then the ConfigMap needs to be named as istio-shared-configmap-asm-1-22. Mesh configuration has to be provided within the data section under mesh.

```bash
 kubectl apply -f configmap.yaml
```
### 5. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.2/cert-manager.yaml
```
#### Validate cert-manager pods

```bash
kubectl get pods -n cert-manager
```
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-c96b66c4c-fh9vw               1/1     Running   0          3h21m
cert-manager-cainjector-54668df7b7-lwhwx   1/1     Running   0          3h21m
cert-manager-webhook-5bdfbff59c-xqlcg      1/1     Running   0          3h21m

```
### 6. Setup cluster-issuer and Certificate resources

Set an email address in cluster-issuer.yaml to register with the ACME server.
```bash
kubectl apply -f certificate.yaml
```
This should create k8s secret bookinfo-certs in aks-istio-ingress namespace as requested by the certificate resource created above.

```bash
kubectl get secret -n aks-istio-ingress
```
```bash
NAME                                                               TYPE                 DATA   AGE
bookinfo-certs                                                     kubernetes.io/tls    2      3h15m
sh.helm.release.v1.asm-igx-aks-istio-ingressgateway-external.v52   helm.sh/release.v1   1      7m42s
sh.helm.release.v1.asm-igx-aks-istio-ingressgateway-external.v53   helm.sh/release.v1   1      2m44s
sh.helm.release.v1.asm-igx-aks-istio-ingressgateway-internal.v53   helm.sh/release.v1   1      7m7s
sh.helm.release.v1.asm-igx-aks-istio-ingressgateway-internal.v54   helm.sh/release.v1   1      2m11s
```
### Validate https request to productpage service

Verify that the productpage can be accessed via the HTTPS endpoint
```bash
curl -sS https://cert-test.online/productpage | grep -o "<title>.*</title>"
```

This should print ``` <title>Simple Bookstore App</title> ```
## License

[MIT](https://choosealicense.com/licenses/mit/)
