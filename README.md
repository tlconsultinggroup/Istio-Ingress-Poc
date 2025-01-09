# Secure ingress gateway for Istio service mesh add-on for Azure Kubernetes Service

The Deploy external Istio Ingress gateway article describes how to configure an external/internal ingress gateway to expose an HTTP service to external/internal traffic. This repo shows how to expose a secure HTTPS service using either simple or mutual TLS.

## Prerequisites
* Set environment variables
* Install Istio add-on
* Enable sidecar injection
* Deploy sample application

#### 1. Set environment variables

```bash
export CLUSTER=<cluster-name>
export RESOURCE_GROUP=<resource-group-name>
export LOCATION=<location>
```
#### 2. Install Istio add-on

To install the Istio add-on when creating the cluster, use the --enable-azure-service-mesh or--enable-asm parameter.

```bash
az group create --name ${RESOURCE_GROUP} --location ${LOCATION}

az aks create --resource-group ${RESOURCE_GROUP} --name ${CLUSTER} --enable-asm --generate-ssh-keys
```
To install mesh for existing cluster
The following example enables Istio add-on for an existing AKS cluster:

```bash
az aks mesh enable --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}
```
To verify the Istio add-on is installed on your cluster, run the following command:

```bash
az aks show --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}  --query 'serviceMeshProfile.mode'
```
Confirm the output shows Istio.

#### 3. Enable sidecar injection

To automatically install sidecar to any new pods, you need to annotate your namespaces with the revision label corresponding to the control plane revision currently installed.

If you're unsure which revision is installed, use:
```bash
az aks show --resource-group ${RESOURCE_GROUP} --name ${CLUSTER}  --query 'serviceMeshProfile.istio.revisions'
```
Apply the revision label:
```bash
kubectl label namespace default istio.io/rev=asm-X-Y
```
#### 3.1 Trigger sidecar injection

You can either deploy the sample application provided for testing, or trigger sidecar injection for existing workloads.

If you have existing applications to be added to the mesh, ensure their namespaces are labeled as in the previous step, and then restart their deployments to trigger sidecar injection:
```bash
kubectl rollout restart -n <namespace> <deployment name>
```
Verify that sidecar injection succeeded by ensuring all containers are ready and looking for the istio-proxy container in the kubectl describe output, for example:

```bash
kubectl describe pod -n namespace <pod name>
```
The istio-proxy container is the Envoy sidecar. Your application is now part of the data plane

#### Deploy sample application

We have deployed sample application that uses OpenAI on AKS to populate a few microservices. This sample application consists of Kubernetes deployments and services that need to demonstrate Istio functionality.

Please refer to the aks-store-all-in-one.yaml file and [steps](https://learn.microsoft.com/en-au/azure/aks/open-ai-quickstart?tabs=aoai) from Azure documentation. 

### Enable external ingress gateway

Use az aks mesh enable-ingress-gateway to enable an externally accessible Istio ingress on your AKS cluster:

```bash
az aks mesh enable-ingress-gateway --resource-group $RESOURCE_GROUP --name $CLUSTER --ingress-gateway-type external
```

Use kubectl get svc to check the service mapped to the ingress gateway:

```bash
kubectl get svc aks-istio-ingressgateway-external -n aks-istio-ingress
```
Observe from the output that the external IP address of the service is a publicly accessible one:

```bash
NAME                                TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                      AGE
aks-istio-ingressgateway-external   LoadBalancer   10.0.10.249   <EXTERNAL_IP>   15021:30705/TCP,80:32444/TCP,443:31728/TCP   4m21s
```
Applications aren't accessible from outside the cluster by default after enabling the ingress gateway. To make an application accessible, map the sample deployment's ingress to the Istio ingress gateway using the following manifest:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: store-admin-gateway-external
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: store-admin-vs-external
spec:
  hosts:
  - "*"
  gateways:
  - store-admin-gateway-external
  http:
    route:
    - destination:
        host: store-admin
        port:
          number: 80
EOF
```
#### Note

The selector used in the Gateway object points to istio: aks-istio-ingressgateway-external, which can be found as label on the service mapped to the external ingress that was enabled earlier.

Set environment variables for external ingress host and ports:

```bash
export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export GATEWAY_URL_EXTERNAL=$INGRESS_HOST_EXTERNAL:$INGRESS_PORT_EXTERNAL
```
Retrieve the external address of the sample application:
```bash
echo "http://$GATEWAY_URL_EXTERNAL/"
```

Navigate to the URL from the output of the previous command and confirm that the sample application's product page is displayed.


### Enable internal ingress gateway

Use az aks mesh enable-ingress-gateway to enable an internal Istio ingress on your AKS cluster:

```bash
az aks mesh enable-ingress-gateway --resource-group $RESOURCE_GROUP --name $CLUSTER --ingress-gateway-type internal
```
Use kubectl get svc to check the service mapped to the ingress gateway:

```bash
kubectl get svc aks-istio-ingressgateway-internal -n aks-istio-ingress
```
Observe from the output that the external IP address of the service isn't a publicly accessible one and is instead only locally accessible:

```bash
NAME                                TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                                      AGE
aks-istio-ingressgateway-internal   LoadBalancer   10.0.182.240  <IP>      15021:30764/TCP,80:32186/TCP,443:31713/TCP   87s
```
After enabling the ingress gateway, applications need to be exposed through the gateway and routing rules need to be configured accordingly. Use the following manifest to map the sample deployment's ingress to the Istio ingress gateway:

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: default
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - hosts:
    - store.tlcaksmtlsdemo.io
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      credentialName: storepage-credential
      mode: MUTUAL
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: store-vs-external
  namespace: default
spec:
  gateways:
  - bookinfo-gateway
  hosts:
  - store.tlcaksmtlsdemo.io
  http:
  - route:
    - destination:
        host: store-admin
        port:
          number: 80
EOF
```
## Secure ingress gateway for Istio service mesh with AKS cluster

The Deploy external  Istio Ingress gateway  describes how to configure an ingress gateway to expose an HTTP service to external/internal traffic. This article shows how to expose a secure HTTPS service mutual TLS.

### Required client/server certificates and keys

We used openssl to create certificates and keys. 

1. Create a root certificate and private key for signing the certificates for sample services:

```bash
mkdir bookinfo_certs
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=store info Inc./CN=tlcaksmtlsdemo.io' -keyout bookinfo_certs/bookinfo.com.key -out bookinfo_certs/bookinfo.com.crt
```
2. Generate a certificate and private key for productpage.bookinfo.com:

```bash
openssl req -out bookinfo_certs/productpage.bookinfo.com.csr -newkey rsa:2048 -nodes -keyout bookinfo_certs/productpage.bookinfo.com.key -subj "/CN=storepage.tlcaksmtlsdemo.io/O=product organization"
openssl x509 -req -sha256 -days 365 -CA bookinfo_certs/bookinfo.com.crt -CAkey bookinfo_certs/bookinfo.com.key -set_serial 0 -in bookinfo_certs/productpage.bookinfo.com.csr -out bookinfo_certs/productpage.bookinfo.com.crt
```
3. Generate a client certificate and private key:
```bash
openssl req -out bookinfo_certs/client.bookinfo.com.csr -newkey rsa:2048 -nodes -keyout bookinfo_certs/client.bookinfo.com.key -subj "/CN=client.tlcaksmtlsdemo.io/O=client organization"
openssl x509 -req -sha256 -days 365 -CA bookinfo_certs/bookinfo.com.crt -CAkey bookinfo_certs/bookinfo.com.key -set_serial 1 -in bookinfo_certs/client.bookinfo.com.csr -out bookinfo_certs/client.bookinfo.com.crt
```

### Configure a TLS ingress gateway

Create a Kubernetes TLS secret for the ingress gateway; use Azure Key Vault to host certificates/keys and Azure Key Vault Secrets Provider add-on to sync secrets to the cluster.

#### Set up Azure Key Vault and sync secrets to the cluster

1. Create Azure Key Vault

It is required to have an Azure Key Vault resource to supply the certificate and key inputs to the Istio add-on.
```bash
export AKV_NAME=<azure-key-vault-resource-name>  
az keyvault create --name $AKV_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
```
2. Enable Azure Key Vault provider for Secret Store CSI Driver add-on on your cluster.

```bash
az aks enable-addons --addons azure-keyvault-secrets-provider --resource-group $RESOURCE_GROUP --name $CLUSTER
```
3. Authorize the user-assigned managed identity of the add-on to access Azure Key Vault resource using access policy. Alternatively, if your Key Vault is using Azure RBAC for the permissions model, follow the instructions here to assign an Azure role of Key Vault for the add-on's user-assigned managed identity.

```bash
OBJECT_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.objectId' -o tsv | tr -d '\r')
CLIENT_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER --query 'addonProfiles.azureKeyvaultSecretsProvider.identity.clientId')
TENANT_ID=$(az keyvault show --resource-group $RESOURCE_GROUP --name $AKV_NAME --query 'properties.tenantId')

az keyvault set-policy --name $AKV_NAME --object-id $OBJECT_ID --secret-permissions get list
```

4. Create secrets in Azure Key Vault using the certificates and keys.

```bash
az keyvault secret set --vault-name $AKV_NAME --name test-productpage-bookinfo-key --file bookinfo_certs/productpage.bookinfo.com.key
az keyvault secret set --vault-name $AKV_NAME --name test-productpage-bookinfo-crt --file bookinfo_certs/productpage.bookinfo.com.crt
az keyvault secret set --vault-name $AKV_NAME --name test-bookinfo-crt --file bookinfo_certs/bookinfo.com.crt
```
5. Please make sure to use the following manifest to deploy SecretProviderClass to provide Azure Key Vault-specific parameters to the CSI driver.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: storepage-credential-spc
  namespace: aks-istio-ingress
spec:
  parameters:
    cloudName: ""
    keyvaultName: $AKV_NAME
    objects: |
      array:
        - |
          objectName: test-storepage-bookinfo-key
          objectType: secret
          objectAlias: "test-storepage-bookinfo-key"
        - |
          objectName: test-storepage-bookinfo-crt
          objectType: secret
          objectAlias: "test-storepage-bookinfo-crt"
        - |
          objectName: test-storeinfo-crt
          objectType: secret
          objectAlias: "test-storeinfo-crt"
    tenantId: $TENANT_ID
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $CLIENT_ID 
  provider: azure
  secretObjects:
  - data:
    - key: tls.key
      objectName: test-storepage-bookinfo-key
    - key: tls.crt
      objectName: test-storepage-bookinfo-crt
    - key: ca.crt
      objectName: test-storeinfo-crt
    secretName: storepage-credential
    type: opaque
EOF
```
6. Use the following manifest to deploy a sample pod. The secret store CSI driver requires a pod to reference the SecretProviderClass resource to ensure secrets sync from Azure Key Vault to the cluster.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secrets-store-sync-productpage
  namespace: aks-istio-ingress
spec:
  containers:
    - name: busybox
      image: mcr.microsoft.com/oss/busybox/busybox:1.33.1
      command:
        - "/bin/sleep"
        - "10"
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "storepage-credential-spc"
EOF
```
* Verify storepage-credential secret created on the cluster namespace aks-istio-ingress as defined in the SecretProviderClass resource.

```bash
kubectl describe secret/storepage-credential -n aks-istio-ingress
```
Example output:
```bash
Name:         storepage-credential
Namespace:    aks-istio-ingress
Labels:       secrets-store.csi.k8s.io/managed=true
Annotations:  <none>

Type:  tls

Data
====
cert:  1066 bytes
key:   1704 bytes
```
7. Configure ingress gateway and virtual service

Route HTTPS traffic via the Istio ingress gateway to the sample applications. Use the following manifest to deploy gateway and virtual service resources.
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: bookinfo-gateway
  namespace: default
spec:
  selector:
    istio: aks-istio-ingressgateway-external
  servers:
  - hosts:
    - store.tlcaksmtlsdemo.io
    port:
      name: https
      number: 443
      protocol: HTTPS
    tls:
      credentialName: storepage-credential
      mode: MUTUAL
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: store-vs-external
  namespace: default
spec:
  gateways:
  - bookinfo-gateway
  hosts:
  - store.tlcaksmtlsdemo.io
  http:
  - route:
    - destination:
        host: store-admin
        port:
          number: 80
EOF
```
8. Test the Service by using the below Curl

```bash
export INGRESS_HOST_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export SECURE_INGRESS_PORT_EXTERNAL=$(kubectl -n aks-istio-ingress get service aks-istio-ingressgateway-external -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export SECURE_GATEWAY_URL_EXTERNAL=$INGRESS_HOST_EXTERNAL:$SECURE_INGRESS_PORT_EXTERNAL

curl -s -HHost:store.tlcaksmtlsdemo.io --resolve "store.tlcaksmtlsdemo.io:$SECURE_INGRESS_PORT_EXTERNAL:$INGRESS_HOST_EXTERNAL" --cacert bookinfo_certs/bookinfo.com.crt --cert bookinfo_certs/client.bookinfo.com.crt --key bookinfo_certs/client.bookinfo.com.key "https://store.tlcaksmtlsdemo.io:$SECURE_INGRESS_PORT_EXTERNAL/products"

![image](https://github.com/user-attachments/assets/5a41182f-ba7a-48b4-9fef-7a99d41a5df5)

```

Note: The server uses the CA certificate to verify its clients, and we must use the key ca.crt to hold the CA certificate.

Enabling mTLS for E-W communication
```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: mtls-pa-default
  namespace: default
spec:
  mtls:
    mode: STRICT
EOF
```
### Configure a mutual TLS ingress gateway

Extend your gateway definition to support mutual TLS.

1. Update the ingress gateway credential by deleting the current secret and creating a new one. The server uses the CA certificate to verify its clients, and we must use the key ca.crt to hold the CA certificate.

```bash
kubectl delete secretproviderclass storepage-credential-spc -n aks-istio-ingress
kubectl delete secret/storepage-credential -n aks-istio-ingress
kubectl delete pod/secrets-store-sync-storepage -n aks-istio-ingress
```
Use the following manifest to recreate SecretProviderClass with CA certificate.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: storepage-credential-spc 
  namespace: aks-istio-ingress
spec:
  provider: azure
  secretObjects:
  - secretName: productpage-credential
    type: opaque
    data:
    - objectName: test-storepage-bookinfo-key
      key: tls.key
    - objectName: test-storepage-bookinfo-crt
      key: tls.crt
    - objectName: test-storeinfo-crt
      key: ca.crt
  parameters:
    useVMManagedIdentity: "true"
    userAssignedIdentityID: $CLIENT_ID 
    keyvaultName: $AKV_NAME
    cloudName: ""
    objects:  |
      array:
        - |
          objectName: test-storepage-bookinfo-key
          objectType: secret
          objectAlias: "test-storepage-bookinfo-key"
        - |
          objectName: test-storepage-bookinfo-crt
          objectType: secret
          objectAlias: "test-storepage-bookinfo-crt"
        - |
          objectName: test-storeinfo-crt
          objectType: secret
          objectAlias: "test-bookinfo-crt"
    tenantId: $TENANT_ID
EOF

```
Use the following manifest to redeploy sample pod to sync secrets from Azure Key Vault to the cluster.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secrets-store-sync-productpage
  namespace: aks-istio-ingress
spec:
  containers:
    - name: busybox
      image: registry.k8s.io/e2e-test-images/busybox:1.29-4
      command:
        - "/bin/sleep"
        - "10"
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "storepage-credential-spc "
EOF
```

* Verify productpage-credential secret created on the cluster namespace aks-istio-ingress.
```bash
kubectl describe secret/storepage-credential -n aks-istio-ingress
```
Example output:

```bash
Name:         storepage-credential
Namespace:    aks-istio-ingress
Labels:       secrets-store.csi.k8s.io/managed=true
Annotations:  <none>

Type:  opaque

Data
====
ca.crt:   1188 bytes
tls.crt:  1066 bytes
tls.key:  1704 bytes
```
2. Use the following manifest to update the gateway definition to set the TLS mode to MUTUAL.
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: aks-istio-ingressgateway-external # use istio default ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: MUTUAL
      credentialName: storepage-credential # must be the same as secret
    hosts:
    - store.tlcaksmtlsdemo.io
EOF
```
