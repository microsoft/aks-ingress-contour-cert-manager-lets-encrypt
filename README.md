# AKS Ingress with Contour and Let's Encrypt

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

  ```bash

  az aks create \
  -g $LAB_AKS_RG \
  -n $LAB_AKS_NAME \
  --node-resource-group $LAB_AKS_NODE_RG \
  --generate-ssh-keys \
  --enable-managed-identity \
  --node-count 1

  ```

- Install the AKS CLI

  ```bash

  sudo az aks install-cli

  ```

- Login to the AKS Cluster

  ```bash

  az aks get-credentials -g $LAB_AKS_RG -n $LAB_AKS_NAME

  ```

- List the pods on the cluster

  ```bash

  kubectl get pods -A

  ```

## Setup Contour

- Apply the Contour Kustomization

    ```bash

    # due to timing, the let's encrypt setup sometimes fails the first time
    # wait a few seconds and rerun the command
    kubectl apply -k deploy/contour

    ```

- Check Contour Deployment

  ```bash

  kubectl get pods -n projectcontour

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
  - You may need to retry due to the acme handshake
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

- Check the endpoint
  - Result should be 301

    ```bash

    http http://$LAB_FQDN/benchmark/17

    ```

- Check the https endpoint
  - Result should be 200
    - 0123456789ABCDEF0

    ```bash

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
