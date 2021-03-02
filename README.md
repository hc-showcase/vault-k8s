# Integrate a Kubernetes Cluster with an External Vault with Vault already prepared

This demo showcases how applicationis running on Kubernetes can consume secrets from Vault. The demo application has two paths: /api and /env. /api works in all three demos, /env only in the last one.

## Prerequisites
* Vault is already up and running. By default the applciation expects the root token to be 'root'.
* A KV secrets engine is configured.
* A secret exists under the path /secret/webapp/config with two K/V pair: username and password. Values don't matter.
* A Kubernetes cluster is available. Or use minikube.

## (Optional) Build the demo application
The folder 'webapp-build' contains the application as well as the Dockerfile to package the application as a container image. This step is optional as the container image is available on Docker Hub: mskaesz/webapp-ruby

Build the container:
```
cd <repo>/webapp-build
docker build -t mskaesz/webapp-ruby:latest .
```

Push the container to a registry that your Kubernetes cluster can access:
```
docker push mskaesz/webapp-ruby:latest
```

## Deploy application with hard-coded Vault address
The application accesses Vault directly via API.

```
cd <repo>/webapp-directly
kubectl apply -f webapp-directly.yaml
kubectl port-forward <pod name> 8080:8080 (only needed if do not have ingress)
curl localhost:8080/api
```

You should see something like '{"password"=>"43", "username"=>"user0815"}'


# Deploy service and endpoints to access an external Vault
The application accesses Vault directly via API, but uses a Kubernetes service to lookup the Vault address.

Deploy the service:
```
cd <repo>/webapp-through-service
kubectl apply -f service-and-endpoint.yaml
kubectl apply -f webapp-through-service.yaml
kubectl port-forward <pod name> 8080:8080 (only needed if you do not have ingress)
curl localhost:8080/api
```

## Access secrets via Vault Agent
The application access the secrets via enviroment variable. THe Vault agent is running as a sidecar next to the application.

### Install the Vault Helm chart configured to address an external Vault

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
    --set "injector.externalVaultAddr=http://external-vault:8200"
```

### Configure Kubernets Authentication with Vault

See https://learn.hashicorp.com/tutorials/vault/kubernetes-external-vault?in=vault/kubernetes

Add the Vault policy to Vault:
```
cd <repo>/webapp-through-service-and-agent
vault policy write webapp webapp.policy
```

Configure a role:
```
vault write auth/kubernetes/role/webapp \
        bound_service_account_names=internal-app \
        bound_service_account_namespaces=default \
        policies=webapp \
        ttl=24h
```

Deploy the service account and application:
```
kubectl apply -f service-account-internal-app.yaml 
kubectl apply -f webapp-through-service-and-agent.yaml 
```

Add the Vault agent:
```
kubectl patch deployment webapp-through-service-and-agent --patch-file patch-deployment-agent.yaml
```

Consume a secret:
```
kubectl patch deployment webapp-through-service-and-agent --patch-file patch-deployment-credentials.yaml
kubectl exec -it <pod-name> -c app -- cat /vault/secrets/credentials
kubectl port-forward <pod name> 8080:8080 (only needed if you do not have ingress)
curl localhost:8080/env
```
