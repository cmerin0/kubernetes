# Cheat Sheet of Kubernetes
> This is a cheat sheet to gather all the useful commands to use in Kubernetes, also to add examples of use of most of the technologies and their explanation.

## Useful files for Kubernets functioning

### kube-apiserver

### kube-config

### Role, RoleBinding, ClusterBinding, ClusterRoleBinding

## Context and Namespaces 

> kubectl config view -> View kubeconfig.
> kubectl config use-context <context_name>: -> Switch context
> kubectl config set-context --current --namespace=<namespace_name> -> Set default namespace for current context.
> kubectl get all -n <namespace> -> Get all resources in a specific namespace.

## Set Up User Authentication

### Generate a private key and CSR
> openssl genrsa -out username.key 2048
> openssl req -new -key username.key -subj "/CN=username" -out username.csr

### Sign the CSR with the cluster CA
> openssl x509 -req -in username.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -out username.crt -CAcreateserial

### Configure kubeconfig
> kubectl config set-credentials username --client-certificate=username.crt --client-key=username.key
> kubectl config set-context developer --cluster=kubernetes --user=username
> kubectl config use-context developer

## Set Up RBAC with imperative commands 
1. Create a Role with full permissions on pods, services, and PVCs
> kubectl create role developer-role --resource=pods,services,persistentvolumeclaims --verb="*" -n development

2. Bind the Role to Username
> kubectl create rolebinding developer-rolebinding --role=developer-role --user=username -n development
