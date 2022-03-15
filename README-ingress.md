# Enable Ingress Controller

## HTTPS Ingress Controller on Azure Cloud

[Learn about ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/)


Azure Load Balancer is of type L4 (transport-level) meaning they do not understand HTTP which is at the application level (L7) and therefore cannot be configured as a TLS termination point

A possible solution could be that you deploy a [ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/). In this document, we will describe two types of possible solutions. Choose one of them.


* Ingress controller Using NGINX (Option 1)
* Application Gateway Ingress Controller (AGIC) (Option 2)


Ingress controller Using NGINX (Option 1)
-----------------------------------------
Follow the instructions [here](https://docs.microsoft.com/en-us/azure/aks/ingress-tls?tabs=azure-cli#import-the-images-used-by-the-helm-chart-into-your-acr). This will import image and set variables required.


### Install ingress by running the following commands
```
❯ DNS_LABEL=test-stardog
❯ ACR_URL=$REGISTRY_NAME.azurecr.io
❯ kubectl create namespace ingress
❯ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
❯ helm install nginx-ingress ingress-nginx/ingress-nginx \
--namespace ingress \
--version 3.36.0 \
--set controller.replicaCount=2 \
--set controller.nodeSelector."kubernetes\.io/os"=linux \
--set controller.image.registry=$ACR_URL \
--set controller.image.image=$CONTROLLER_IMAGE \
--set controller.image.tag=$CONTROLLER_TAG \
--set controller.image.digest="" \
--set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
--set controller.admissionWebhooks.patch.image.registry=$ACR_URL \
--set controller.admissionWebhooks.patch.image.image=$PATCH_IMAGE \
--set controller.admissionWebhooks.patch.image.tag=$PATCH_TAG \
--set controller.service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$DNS_LABEL \
--set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
--set defaultBackend.image.registry=$ACR_URL \
--set defaultBackend.image.image=$DEFAULTBACKEND_IMAGE \
--set defaultBackend.image.tag=$DEFAULTBACKEND_TAG
```

Application Gateway Ingress Controller (AGIC) (Option 2)
---------------------------------------------------------

You can enable the Application Gateway Ingress Controller (AGIC) AKS add-on.
There are several advantages to using AGIC such as but not limited to
* It’s a fully managed service
* It has integration with other Azure services.

There are also disadvantages such as but not limited to
* Regular expression is not supported in ingress rules
    * Rewrite
    * Path routing

Using Letsencrypt
* Currently it not possible to used letsencrypt as the Stardog service require rewriting capability to work.

### Create an Application Gateway (if needed)

```
❯ ID="REPLACE_PROJECT_NAME"
❯ az network public-ip create -n ${ID}PublicIP -g ${ID}Group --allocation-method Static --sku Standard --dns-name ${ID}

❯ az network vnet create -n ${ID}Vnet -g ${ID}Group --address-prefix 11.0.0.0/8 --subnet-name ${ID}Subnet --subnet-prefix 11.1.0.0/16

❯ az network application-gateway create -n ${ID}ApplicationGateway -l eastus2 -g ${ID}Group --sku Standard_v2 --public-ip-address ${ID}PublicIP --vnet-name ${ID}Vnet --subnet ${ID}Subnet
```

### Enable AGIC

```
❯ appgwId=$(az network application-gateway show -n ${ID}ApplicationGateway -g ${ID}Group -o tsv --query "id")

❯ az aks enable-addons -n $AKS_NAME -g ${ID}Group -a ingress-appgw --appgw-id $appgwId
```
### Peer AKS and AGIC together

```
❯ nodeResourceGroup=$(az aks show -n $AKS_NAME -g ${ID}Group -o tsv --query "nodeResourceGroup") ❯ aksVnetName=$(az network vnet list -g $nodeResourceGroup -o tsv --query "[0].name")

❯ aksVnetId=$(az network vnet show -n $aksVnetName -g $nodeResourceGroup -o tsv --query "id")

❯ az network vnet peering create -n ${ID}AppGWtoAKSVnetPeering -g ${ID}Group --vnet-name ${ID}Vnet --remote-vnet $aksVnetId --allow-vnet-access

❯ appGWVnetId=$(az network vnet show -n ${ID}Vnet -g ${ID}Group -o tsv --query "id")

❯ az network vnet peering create -n ${ID}AKStoAppGWVnetPeering -g $nodeResourceGroup --vnet-name $aksVnetName --remote-vnet $appGWVnetId --allow-vnet-access
```

For more detail please see the official documentation [Tutorial: Enable Application Gateway Ingress Controller add-on for an existing AKS cluster with an existing Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-existing?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Faks%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json)


## Using self-signed certificate.

### Create our self-signed certificate
(if you have a certificate, this step is not needed)

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out aks-ingress-tls.crt -keyout aks-ingress-tls.key -subj "/CN=test-project.eastus2.cloudapp.azure.com/O=aks-ingress-tls"
```

#### Register our certificate with k8s in the proper namespace.

```
❯ kubectl create namespace stardog
❯ kubectl create secret tls stardog-tls-secret --namespace stardog --key aks-ingress-tls.key --cert aks-ingress-tls.crt
```

### Configure value.yaml file configuration for ingress controller

Add to your custom value.yaml file and set the values as you need.

```
service:
    type: ClusterIP

ingress:
  enabled: true
  annotations: 
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/app-root: "/admin/alive"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
  className: nginx # Could be nginx or azure/application-gateway
  hosts:
    - host: {ADD_HOST_NAME}
      paths:
        - path: /(.*)
  tls: 
   - secretName: stardog-tls-secret
     hosts:
       - {ADD_HOST_NAME}
```

## Using cert-manager

The NGINX ingress controller supports TLS termination. There are several ways to retrieve and configure certificates for HTTPS. This article demonstrates using cert-manager, which provides automatic Let's Encrypt certificate generation and management functionality.

Follow the instructions [here](https://docs.microsoft.com/en-us/azure/aks/ingress-tls?tabs=azure-cli#import-the-images-used-by-the-helm-chart-into-your-acr). This will import image and set variables required. To install the cert-manager controller:

```
helm repo add jetstack https://charts.jetstack.io

# Label the ingress namespace to disable resource validation

kubectl label namespace ingress cert-manager.io/disable-validation=true --overwrite

helm install cert-manager jetstack/cert-manager \
    --namespace ingress\
    --version $CERT_MANAGER_TAG \
    --set installCRDs=true \
    --set nodeSelector."kubernetes\.io/os"=linux \
    --set image.repository=$ACR_URL/$CERT_MANAGER_IMAGE_CONTROLLER \
    --set image.tag=$CERT_MANAGER_TAG \
    --set webhook.image.repository=$ACR_URL/$CERT_MANAGER_IMAGE_WEBHOOK \
    --set webhook.image.tag=$CERT_MANAGER_TAG \
    --set cainjector.image.repository=$ACR_URL/$CERT_MANAGER_IMAGE_CAINJECTOR \
    --set cainjector.image.tag=$CERT_MANAGER_TAG

```
### Configure cluster issuers

If you want to use Let’s encrypt

```
#file letencrypt.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: CUSTOM-EMAIL@MAIL.com
    privateKeySecretRef:
      name: letsencrypt
    solvers:
    - http01:
        ingress:
          class: nginx
          podTemplate:
            spec:
              nodeSelector:
                "kubernetes.io/os": linux
```
Could be deployed using: \
`kubectl apply -f letsencrypt.yaml`

### Or use self-signed Certificate

```
#file: selfsigned.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
```
Could be deployed using: \
`kubectl apply -f selfsigned.yaml`


Add to your custom value.yaml file and set the values as you need.

```
service:
   type: ClusterIP

ingress:
  enabled: true
  annotations: 
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/app-root: "/admin/alive"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    cert-manager.io/cluster-issuer: letsencrypt # Could be letsencrypt or selfsigned
  className: nginx # Could be nginx or azure/application-gateway
  hosts:
    - host: {ADD_HOST_NAME}
      paths:
        - path: /(.*)
  tls: 
   - secretName: stardog-tls-secret
     hosts:
       - {ADD_HOST_NAME}
```

Deploy Stardog
--------------
```
❯ kubectl -n stardog create secret generic stardog-license --from-file stardog-license-key.bin=../stardog-license.bin

❯ helm install --values {DIR/CUSTOM_VALUE_FILE.yaml} --namespace stardog --timeout 30m {INSTALLATION_NAME} stardog/stardog

```