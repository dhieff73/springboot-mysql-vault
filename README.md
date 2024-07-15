# Spring Boot Application Deployment on Kubernetes with MySQL and Vault 
This repository contains the necessary files and instructions to deploy a Spring Boot application with MySQL on a Kubernetes cluster. The database configuration is managed as secrets in Vault and injected into the Spring Boot pod.

## Table of contents
#### [1. Prerequisites](README.md)
#### [2. Clone the Repository](README.md)
#### [3. Setup Vault](README.md) 
#### [4. Setup Vault Agent Injector and create secrets (database config)](README.md) 
#### [5. Build and Push Docker Image](README.md)
#### [6. create the spring boot deployment with secrets injection](README.md) 



## Prerequisites
- kuberentes cluster
- docker


## Clone the Repository
`git clone https://github.com/dhieff73/springboot-mysql-vault.git`

## Setup Vault 
First you must to change dircetory to vault-manifests 
`cd vault-manifests`

#### Create ServiceAccount 
The Vault server requires additional Kubernetes permissions to perform its operations. Therefore, a ClusterRole (with the appropriate permissions) must be assigned to a ServiceAccount via a ClusterRoleBinding.
So to create the service account we have to apply the file rbac.yaml 

`kubectl apply -f rbac.yaml`

#### Create ConfigMap
ConfigMaps in Kubernetes allow us to mount files onto containers without needing to modify the Dockerfile or rebuild the container image. This feature is extremely useful in cases where configurations need to be changed or created via files. Vault requires a configuration file with the appropriate parameters to start its servers.

`kubectl apply -f configmap.yaml`


#### Deploy vault services 
For the Vault server, we will create a headless service for internal use. This will be very useful when we scale Vault to multiple replicas. A non-headless service will be created for the user interface because we want to balance requests to the replicas when accessing the user interface.
Vault exposes its user interface on port 8200. We will use a non-headless service of type NodePort because we want to access this endpoint from outside the Kubernetes cluster.

`kubectl apply -f services.yaml`

**PublishNotReadyAddresses:**

The publishNotReadyAddresses option changes this behavior by including pods that may or may not be in the "ready" state.

#### Create vault deployment 

`kubectl aaply -f vault-deploy.yaml`

#### Unseal and initialize vault 
Creation of tokens and keys to start using Vault.


`kubectl exec <pod-name> -- vault operator init -key-shares=1 -key-threshold=1 -format=json > keys.json`

`VAULT_UNSEAL_KEY=$(cat keys.json | jq -r ".unseal_keys_b64[]")`

`echo $VAULT_UNSEAL_KEY`

`VAULT_ROOT_KEY=$(cat keys.json | jq -r ".root_token")`

`echo $VAULT_ROOT_KEY`

`kubectl exec <pod-name>-- vault operator unseal $VAULT_UNSEAL_KEY`

`kubectl exec <pod-name>-- vault login $VAULT_ROOT_KEY`

The result must look like this: 

![image](https://github.com/user-attachments/assets/ef110dab-db15-4747-a2b2-da8c632a66c3)


Since we have created a service on NodePort 32000, we will be able to access the Vault user interface using any of the nodes' IP addresses on port 32000. You can use the saved token to log in to the user interface.
![image](https://github.com/user-attachments/assets/71262211-a15c-4c20-b689-8f6243ad69bd)


## Setup Vault Agent Injector and create secrets (database config)

We have to change dircetory to vault-injector-manifests 

`cd vault-injector-manifests`

#### Create ServiceAccount 


` kubectl apply -f rbac.yaml`

#### Create MutatingWebhook 

This webhook is responsible for intercepting and sending pod events to the injector controller. It sends the webhook to the service endpoint of the injector controller at the path /mutate.

`kubectl apply -f mutating-webhook.yaml`

#### Deploy agent injector 

` kubectl apply -f deployment.yaml`

#### Create service 

` kubectl apply -f service.yaml`

#### Create secrets and policy 

Create secrets under **kv/dev/apps/spring**

`vault kv put kv/dev/apps/spring MYSQL_HOST="10.233.11.126" MYSQL_USER="root" MYSQL_PASSWORD="root" MYSQL_PORT="3306" MYSQL_DATABASE="spring-kube"`

Create a Vault policy named svc-policy that allows read operations on secrets under the path kv/data/dev/apps/spring:

`
vault policy write svc-policy - <<EOH
path "kv/data/dev/apps/spring" {
  capabilities = ["read"]
}
EOH
`
Activate kubernetes authentification: 

` vault auth enable kubernetes`

`vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://kubernetes.default.svc.cluster.local" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    issuer="https://kubernetes.default.svc.cluster.local"
`

Create a vault role with the name **webapp**: 

`vault write auth/kubernetes/role/webapp \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=svc-policy \
        ttl=72h
`

Create a service account named **vault-auth** 

`kubectl create serviceaccount vault-auth`























