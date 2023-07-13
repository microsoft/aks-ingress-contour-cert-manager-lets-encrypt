# AKS Ingress with Contour, Cert-Manager and Let's Encrypt

This lab is designed to help you securely expose your Kubernetes services over HTTPS and deploy multiple applications using `path-based` and `host-based` routing. This can be useful if you have multiple applications running on your cluster and want to ensure that each one is accessible via its own unique and HTTPS secured URL. This lab uses:

- [AKS Cluster](https://learn.microsoft.com/en-us/azure/aks/): Azure Kubernetes Service to deploy and manage cloud native applications in Azure
- [Contour](https://projectcontour.io/): An ingress controller for Kubernetes that works by deploying the Envoy proxy as a reverse proxy and load balancer
- [Let's Encrypt](https://letsencrypt.org/about/): A Certificate Authority (CA) to get a certificate for your domain
- [cert-manager](https://cert-manager.io/docs/): A Certificate Controller to provision and manage TLS certifications from `Let's Encrypt` or any other issuer

## Goals of the Lab

- Enable Contour and Let's Encrypt for secure ingress to an AKS cluster
- Deploy an application with path-based routing
- Deploy a second application with path-based routing
- Deploy an application with host-based routing
- Deploy a second application with host-based routing

## Prerequisites

- A public GitHub account
- An Azure subscription
- A domain name and DNS server
  - Instructions for using Azure DNS are included

## Create a Codespace

- Fork this repo
- Create a Codespace from the forked repo (not this repo)
  - Use the Codespace terminal to work through the lab

## Getting Started

- Set AKS Name

  ```bash

  # edit this value (optional)
  export LAB_AKS_NAME=lab-aks

  ```

- Set Location and Resource Group names

  ```bash

  export LAB_LOCATION=eastus
  export LAB_AKS_RG=${LAB_AKS_NAME}-rg
  export LAB_AKS_NODE_RG=${LAB_AKS_NAME}-node-rg

  ```

- Set DNS Information

  ``` bash

  # edit this value
  export LAB_DNS_ZONE=aks-demo.com

  # if you have an Azure DNS zone, edit this value
  export LAB_DNS_RG=tld

  ```

  ```bash

  export LAB_DNS_HOST=lab
  export LAB_FQDN=$LAB_DNS_HOST.$LAB_DNS_ZONE

  ```

- Check environment variables

  ```bash

  env | grep ^LAB_

  ```

- Update the ingress manifest files

  ```bash

  # replace the FQDN in the manifest files
  find deploy -type f -exec sed -i "s|lab.aks-demo.com|$LAB_FQDN|g" {} \;

  ```

- View ingress manifest changes

  ```bash

  git diff

  ```

## Deploy a basic AKS Cluster

- Login to Azure using a device code

  ```bash

  az login --use-device-code

  ```

- Create a Resource Group

  ```bash

  az group create -n $LAB_AKS_RG -l $LAB_LOCATION

  ```

- Create an AKS Cluster
  - This command will create SSH key files in `$HOME/.ssh`
  - Copy your `id_rsa` and `id_rsa.pub` to `$HOME/.ssh` if you want to use existing SSH keys

  ```bash

  az aks create \
  -g $LAB_AKS_RG \
  -n $LAB_AKS_NAME \
  --node-resource-group $LAB_AKS_NODE_RG \
  --generate-ssh-keys \
  --enable-managed-identity \
  --node-count 1

  ```

- Login to the AKS Cluster

  ```bash

  az aks get-credentials -g $LAB_AKS_RG -n $LAB_AKS_NAME

  ```

- Wait for the pods to start
  - Press `ctl-c` once all pods are running

  ```bash

  kubectl get pods --all-namespaces --watch

  ```

## Setup Contour and Cert-Manager

- Apply the Contour Kustomization

    ```bash

    kubectl apply -k deploy/contour

    # wait for pods to start / complete
    kubectl get pods -n projectcontour --watch

    ```

- Apply the Cert-Manager Kustomization

    ```bash

    kubectl apply -k deploy/cert-manager

    # wait for pods to start
    kubectl wait pod --all -n cert-manager --for=condition=ready --timeout 60s

    # check via the CLI
    kubectl cert-manager check api

    ```

- Edit the lets-encrypt manifest
  - Use a valid email address

  ```bash

  code deploy/lets-encrypt/lets-encrypt.yaml

  ```

- Apply the lets-encrypt Kustomization

  ```bash

  kubectl apply -k deploy/lets-encrypt

  # check secrets
  kubectl get secrets -n cert-manager

  ```

## Setup DNS

- Get the Load Balancer public IP

  ```bash

  export LAB_IP=$(kubectl get svc -n projectcontour envoy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

  # check the env vars
  env | grep ^LAB_

  ```

- If using Azure DNS
  - Create a DNS A Record

  ```bash

  az network dns record-set a add-record \
  -g "$LAB_DNS_RG" \
  -z "$LAB_DNS_ZONE" \
  -n "$LAB_DNS_HOST" \
  -a "$LAB_IP" \
  --ttl 10 -o table

  ```

- Check DNS

    ```bash

    # if this doesn't resolve, the rest of the lab will fail
    ping $LAB_DNS_HOST.$LAB_DNS_ZONE

    ```

## Deploy an application using path-based routing

- Heartbeat is a simple application that allows you to retrieve a known set of data from a known endpoint

- Deploy the application kustomization

  ```bash

  kubectl apply -k deploy/heartbeat

  # wait for the heartbeat pod to start
  kubectl get pods -n heartbeat --watch

  ```

- Check the ingress controllers

  ```bash

  kubectl get ingress -A

  ```

- Check the endpoint
  - Result should be 301

    ```bash

    http http://$LAB_FQDN/heartbeat/17

    ```

- Check the https endpoint
  - You may need to retry due to the acme handshake, this can take up to a minute
  - Result should be 200
    - 0123456789ABCDEF0

    ```bash

    http https://$LAB_FQDN/heartbeat/17

    ```

## Deploy a second application using path-based routing

- The ingress uses path-based routing to route to `/benchmark/17`

- Deploy the application kustomization

  ```bash

  kubectl apply -k deploy/benchmark

  # wait for the benchmark pod to start
  kubectl get pods -n benchmark --watch

  ```

- Check the ingress controllers

  ```bash

  kubectl get ingress -A

  ```

- Check the endpoints

    ```bash

    # Result should be 301
    http http://$LAB_FQDN/benchmark/17

    # Result should be 200
    http https://$LAB_FQDN/benchmark/17

    ```

## Deploy an application using host-based routing

- Deploy Redis for the apps

  ```bash

  kubectl apply -k deploy/redis

  ```

- If using Azure DNS
  - Create the DNS entry
  - dogs.lab.your.zone

    ```bash

    az network dns record-set a add-record \
    -g "$LAB_DNS_RG" \
    -z "$LAB_DNS_ZONE" \
    -n "dogs.$LAB_DNS_HOST" \
    -a "$LAB_IP" \
    --ttl 10 -o table

    ```

- Deploy the dogs-cats app

  ```bash

  kubectl apply -k deploy/dogs-cats

  # wait for pod to start
  kubectl get pods -n dogs-cats --watch

  ```

- Check the ingress controllers

  ```bash

  kubectl get ingress -A

  ```

- Check endpoints

  ```bash

  # should return 301
  http http://dogs.$LAB_FQDN/

  # should return 200
  http https://dogs.$LAB_FQDN/

  ```

## Deploy a second application using host-based routing

- If using Azure DNS
  - Create the DNS entry
  - tabs.lab.your.zone

    ```bash

    az network dns record-set a add-record \
    -g "$LAB_DNS_RG" \
    -z "$LAB_DNS_ZONE" \
    -n "tabs.$LAB_DNS_HOST" \
    -a "$LAB_IP" \
    --ttl 10 -o table

    ```

- Deploy the tabs-spaces app

  ```bash

  kubectl apply -k deploy/tabs-spaces

  # wait for pod to start
  kubectl get pods -n tabs-spaces --watch

  ```

- Check the ingress controllers

  ```bash

  kubectl get ingress -A

  ```

- Check endpoints

  ```bash

  # should return 301
  http http://tabs.$LAB_FQDN/

  # should return 200
  http https://tabs.$LAB_FQDN/

  ```

## Cleanup

- Delete the Kubernetes context

  ```bash

  kubectl config delete-context $LAB_AKS_NAME

  ```

- If using Azure DNS
  - Delete the A record(s)

  ```bash

  az network dns record-set a remove-record \
  -g "$LAB_DNS_RG" \
  -z "$LAB_DNS_ZONE" \
  -n "$LAB_DNS_HOST" \
  -a "$LAB_IP" -o table

  az network dns record-set a remove-record \
  -g "$LAB_DNS_RG" \
  -z "$LAB_DNS_ZONE" \
  -n "dogs.$LAB_DNS_HOST" \
  -a "$LAB_IP" -o table

  az network dns record-set a remove-record \
  -g "$LAB_DNS_RG" \
  -z "$LAB_DNS_ZONE" \
  -n "tabs.$LAB_DNS_HOST" \
  -a "$LAB_IP" -o table

  ```

- Delete the Resource Group

  ```bash

  az group delete -y --no-wait -g $LAB_AKS_RG

  ```

## How to get help

For help and questions about using this lab, please use the [GitHub Discussion tab](https://github.com/microsoft/aks-ingress-contour-lets-encrypt/discussions).

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit <https://cla.opensource.microsoft.com>.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
