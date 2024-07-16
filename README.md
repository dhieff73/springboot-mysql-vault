# Spring Boot Application Deployment on Kubernetes with MySQL and Vault 
This repository contains the necessary files and instructions to deploy a Spring Boot application with MySQL on a Kubernetes cluster. The database configuration is managed as secrets in Vault and injected into the Spring Boot pod.

## Table of contents
1. [Prerequisites](#prerequisites)
2. [Clone the Repository](#clone-the-repository)
3. [Setup Vault](#setup-vault)
4. [Setup Vault Agent Injector and Create Secrets (Database Config)](#setup-vault-agent-injector-and-create-secrets-database-config)
5. [Create MYSQL  and  the spring boot deployments and services with secrets injection](#create-mysql-and-the-spring-boot-deployments-and-services-with-secrets-injection)
---

## Prerequisites
- kuberentes cluster
- docker

---

## Clone the Repository
`git clone https://github.com/dhieff73/springboot-mysql-vault.git`

---

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

---

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

---


##  Create MYSQL and the spring boot deployments and services with secrets injection
Now we have to move to app-deploys directory 

`cd app-deploys`

Then we have to create the mysql deployment with the latest docker image available (mysql-deploy.yaml)

`kubectl apply -f mysql-deploy.yaml`

and create the service 

` kubectl apply -f mysql-svc.yaml`

When the MySQL pod is running normally ![mysql running](https://github.com/user-attachments/assets/fa8d8367-e4cf-4cc1-bb6b-160b4ad28d1b) 
 we can create our spring boot deployment with the injector annotations (spring-deploy.yaml) 

 this is the content of the deployment yaml file : 



## spring-deploy.yaml




``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-kube-app
  labels:
    app: spring-kube-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spring-kube-app
  template:
    metadata:
      labels:
        app: spring-kube-app
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'webapp'
        vault.hashicorp.com/agent-pre-populate-only: 'true'
        vault.hashicorp.com/agent-inject-secret-database-config: 'kv/dev/apps/spring'
        # Environment variable export template
        vault.hashicorp.com/agent-inject-template-database-config: |
          {{ with secret "kv/dev/apps/spring" -}}
            export MYSQL_HOST="{{ .Data.data.MYSQL_HOST }}"
            export MYSQL_USER="{{ .Data.data.MYSQL_USER }}"
            export MYSQL_PORT="{{ .Data.data.MYSQL_PORT }}"
            export MYSQL_DATABASE="{{ .Data.data.MYSQL_DATABASE }}"
            export MYSQL_PASSWORD="{{ .Data.data.MYSQL_PASSWORD }}"
          {{- end }}
    spec:
      serviceAccountName: vault-auth
      containers:
        - name: spring-kube-app
          image: khalil73/spring-kube  # Replace with your Docker image name
          command:
            ['sh', '-c']
          args:
            ['source /vault/secrets/database-config && java -jar /app.jar']
          ports:
            - containerPort: 8089  # Port your Spring Boot app listens on inside the container

```

For the deployment of our Spring Boot application, I've set up a Jenkins pipeline that fetches the code from GitLab, runs tests and code analysis, builds a Docker image, and pushes it to Docker Hub. This image is then downloaded to our Spring Boot service. 

This is the script of the jenkins pipeline : 

## Jenkins pipeline 

``` groovy
pipeline {
    agent any 
    
      environment {
        KUBECONFIG = credentials('kubeconfig_id')
    }
    tools {
        maven "M2_HOME"
        jdk "JAVA_HOME"
    }
    stages {
        
         

        
        
        stage('Fetch code') {
            steps {
                git branch: 'main',
                credentialsId: 'fa500f0a-c585-42c9-8a1d-5cc5f8b8b104',
                url: 'https://gitlab.com/dhieff.14/springkube.git'
            }
        }
      
                
        
 
                
        stage('Maven clean') {
            steps {
                script {
                    def repoPath = "LoginKube"
                    sh "cd ${repoPath} && mvn clean"
                }
            }
        }

        stage('Maven package and finalizing') {
            steps {
                script {
                    def repoPath = "LoginKube"
                    sh "cd ${repoPath} && mvn package"
                }
            }
        }
        
        
                
                stage('Code Quality Check via SonarQube') {
                    steps {
                        script {
                            dir('LoginKube') {
                                withSonarQubeEnv(credentialsId: 'jenkins_token', installationName: 'sonar') {
                                    sh 'mvn sonar:sonar'
                                }
                            }
                        }
                    }
        
                    
                }

        stage('Build Docker Image') {
            steps {
            script {
                def repoPath = "LoginKube"
                def imageName = "khalil73/spring-kube:latest"
                sh "cd ${repoPath} && docker build -t ${imageName} ."
                }
            }
        }
        stage('Push Docker Image to Docker Hub') {
            steps {
                script {
                    def imageName = "khalil73/spring-kube:latest"
                    withCredentials([usernamePassword(credentialsId: 'dockerhub_id', usernameVariable: 'DOCKERHUB_USERNAME', passwordVariable: 'DOCKERHUB_PASSWORD')]) {
                        sh "docker login -u ${DOCKERHUB_USERNAME} -p ${DOCKERHUB_PASSWORD}"
                    }
                    sh "docker push ${imageName}"
                }
            }
        }
        stage('Test Kubernetes Connection') {
            steps {
                script {
                    withCredentials([file(credentialsId: 'kubeconfig_id', variable: 'KUBECONFIG_FILE')]) {
                        writeFile file: 'test-k8s-connection.sh', text: '''
                        #!/bin/bash
                        export KUBECONFIG=${KUBECONFIG_FILE}
                        kubectl get nodes
                        result=$?
                        if [ $result -ne 0 ]; then
                            echo "Failed to connect to Kubernetes cluster"
                            exit $result
                        else
                            echo "Successfully connected to Kubernetes cluster"
                        fi
                        '''
                        sh 'chmod +x test-k8s-connection.sh'
                        sh './test-k8s-connection.sh'
                    }
                }
            }
        }
        
        
        
         stage('Apply Kubernetes Deployment') {
             steps {
                withCredentials([string(credentialsId: 'vagrant-ssh-password', variable: 'SSHPASS')]) {
                    sh '''
                            sshpass -p "$SSHPASS" ssh -o StrictHostKeyChecking=no vagrant@192.168.100.10 << EOF 
                            cd
                            ls
                            cd deployments 
                            ls
                            sudo kubectl apply -f test.yaml
                        '''
                }}
        }
        
        
        
        
}
}
``` 

To make this step manually you can apply the deployment yaml file with a simple command 

`kubectl apply -f spring-deploy.yaml` 

Then we create the spring service : 

`kuebctl apply -f spring-svc.yaml`

Finally, we can test our project with Postman by entering the IP address of the worker node where our app is deployed, along with the NodePort IP of the Spring service. For example, mine is **http://192.168.100.11:32271**

![add user](https://github.com/user-attachments/assets/953ff276-d5ec-4bc2-b0e5-00c1e9363cf1)


![login with user](https://github.com/user-attachments/assets/b08d07d0-0d01-43cc-9efe-3819fe1f01a5)




















