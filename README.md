# Kubernetes Cheat Sheet

A quick reference for essential Kubernetes commands, configuration steps, and RBAC examples.

---

## 📁 Useful Files

- **kube-apiserver manifest:**  
  `/etc/kubernetes/manifests/kube-apiserver.yaml`
- **View running kube-apiserver process:**  
  `ps -ef | grep kube-apiserver`
- **Kubeconfig location:**  
  Usually at `~/.kube/config`

---

## 🗂️ Contexts & Namespaces

- View kubeconfig:  
  ```
  kubectl config view
  ```
- Switch context:  
  ```
  kubectl config use-context <context_name>
  ```
- Set default namespace for current context:  
  ```
  kubectl config set-context --current --namespace=<namespace_name>
  ```
- Get all resources in a namespace:  
  ```
  kubectl get all -n <namespace>
  ```

---

## 👤 User Authentication

### 1. Generate Private Key & CSR
```sh
openssl genrsa -out username.key 2048
openssl req -new -key username.key -subj "/CN=username" -out username.csr
```

### 2. Sign CSR with Cluster CA
```sh
openssl x509 -req -in username.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -out username.crt -CAcreateserial
```

### 3. Configure kubeconfig
```sh
kubectl config set-credentials username --client-certificate=username.crt --client-key=username.key
kubectl config set-context developer --cluster=kubernetes --user=username
kubectl config use-context developer
```

---

## 🔐 RBAC Setup (Imperative Commands)

1. **Create a Role with full permissions on pods, services, and PVCs:**
   ```sh
   kubectl create role developer-role --resource=pods,services,persistentvolumeclaims --verb="*" -n development
   ```

2. **Bind the Role to a user:**
   ```sh
   kubectl create rolebinding developer-rolebinding --role=developer-role --user=username -n development
   ```

---

## 🔐 Pods(po) and Multi-Container Pods 

A Pod is the smallest and most fundamental unit in Kubernetes. It represents a single instance of a running process in your cluster and can contain one or more containers. The containers within a Pod are always co-located and co-scheduled on the same node, and they share the same network namespace, IP address, and storage volumes.

1. **Create a Pod imperatively:**

  - Simple command to create a pod in dry-run mode:
    ```sh
      kubectl run my-pod --image=nginx --dry-run=client -o yaml
    ```

  - A more complex command to create a pod: 
    ```sh
      kubectl run my-pod --image=nginx:1.25 --port=80 \
      --restart=Never \
      --labels=app=nginx,env=prod \
      --limits=cpu=500m,memory=512Mi \
      --requests=cpu=250m,memory=256Mi \
      --env="ENV_VAR_NAME=value" \
      --env="ANOTHER_VAR=another_value" \
      --command -- /bin/sh -c "echo Hello Kubernetes!" \
      --namespace=development
    ```

  - More commands to handle pods:
    ```sh
      kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml     # Generate a pod
      kubectl get pods -A --watch -o wide                                     # List Pods (all namespaces, live updates and show more details)
      kubectl describe pod <pod-name>                                         # Describe a pod
      kubectl delete pod <pod-name>                                           # Delete a pod
      kubectl delete pod <pod-name> --force --grace-period=0                  # Force delete a pod if stuck
      kubectl exec -it <pod-name> -- <command>                                # Execute command within a running pod
      kubectl logs <pod-name> -f -c <container>                               # View pod logs (stream logs + c for multicontainer pods)
      kubectl cp <local-path> <pod-name>:<pod-path>                           # Upload files to pod
      kubectl cp <pod-name>:<pod-path> <local-path>                           # Download files from pod
    ```

2. **Multi-Container Pods**
   
  - Init Container Pod: a specialized container that runs and completes before the main application containers start. It is used to set up prerequisites like downloading configurations, waiting for dependencies (e.g., a database to be ready), or initializing data. Once all init containers succeed, the main containers start. If an init container fails, the pod restarts it until it succeeds (unless restartPolicy: Never is set). Init containers run sequentially, making them ideal for setup tasks that must finish before the app runs.

  - Sidecar Pod: runs alongside the main container in the same pod, enhancing its functionality without modifying the primary app. It typically handles auxiliary tasks like logging, monitoring, proxying, or syncing data. Unlike init containers, sidecars run continuously and share the pod’s network and storage, enabling tight integration (e.g., a sidecar shipping logs from the main app’s volume). Sidecars are common in service meshes (e.g., Istio’s Envoy proxy) and observability tools (e.g., Fluentd for logs).

---

## 🔐 Deployments(deploy), Replicasets(rs) and Rollouts   

A Deployment is a higher-level Kubernetes object that provides declarative updates for Pods and ReplicaSets. You describe the desired state of your application in a Deployment manifest, and the Deployment controller works constantly to ensure the actual state matches the desired state.
A Deployment manages a ReplicaSet -> ReplicaSet manages a set of Pods -> Pod contains one or more containers. 

When you create or update a Deployment, it automatically creates a new ReplicaSet. If you update the Deployment's Pod template (e.g., change the container image), a new ReplicaSet is created for the new version and the old one is scaled down. This ensures that you never have to manually touch a ReplicaSet or Pod.

```sh
  kubectl create deployment <name> --image=nginx --replicas=3             # Generate a deployment
  kubectl get deployments                                                 # List Deployments
  kubectl describe deployment <deployment-name>                           # Describe a deployment
  kubectl set image deployment/deploy-name nginx=nginx:1.16.1 --record    # Update deployment
  kubectl scale deployment <deployment-name> --replicas=5                 # Scale deployment manually
```

A rollout is the process of updating a Deployment. Kubernetes has a built-in kubectl rollout command to manage this process.

```sh
  kubectl rollout status deployment/<deployment-name>                     # Check status of a deployment
  kubectl rollout history deployment/<deployment-name>                    # View the history of the deployment's rollouts
  kubectl rollout undo deployment/<deployment-name>                       # Rollback to the previous revision
  kubectl rollout undo deployment/<deployment-name> --to-revision=3       # Rollback to a specific revision (e.g., revision 3)
  kubectl rollout pause deployment/<deployment-name>                      # Pause the rollout to make changes
  kubectl rollout resume deployment/<deployment-name>                     # Resume the rollout
  kubectl rollout restart deployment/<deployment-name>                    # Restart the rollout
```
---

## 🔐 Services and Networking

1. **The kubectl expose command is the main way to create a Service (svc) imperatively:**

  The create service commands will generate the YAML, but they don't automatically link the service to your my-web-app Deployment. To make them work, you would need to add the selector section to the generated YAML file and then apply it. This is why kubectl expose is often preferred for its simplicity—it automatically handles the selector for you.

   ```sh
    # Expose the 'my-web-app' deployment with a ClusterIP Service
    kubectl expose deployment my-web-app --name=my-app-service --port=8080
    kubectl create service clusterip my-app-service --tcp=80:8080 --dry-run=client -o yaml

    # Expose the 'my-web-app' deployment with a NodePort Service
    kubectl expose deployment my-web-app --name=my-app-nodeport --port=8080 --type=NodePort
    kubectl create service nodeport my-app-nodeport --tcp=80:8080 --node-port=30007 --dry-run=client -o yaml

    # Expose the 'my-web-app' deployment with a LoadBalancer Service
    kubectl expose deployment my-web-app --name=my-app-loadbalancer --port=8080 --type=LoadBalancer
    kubectl create service nodeport my-app-nodeport --tcp=80:8080 --node-port=30007 --dry-run=client -o yaml

    # Explore Available Fields
    kubectl explain service                               # Explain the top-level Service fields
    kubectl explain service.spec                          # Explain the 'spec' section of a Service
    kubectl explain service.spec.ports                    # Explain the 'ports' section of a Service's spec

    # Get info about current services
    kubectl get services                                  # Get a list of all services
    kubectl describe service my-app-service               # Get detailed information about a specific service
    kubectl get services -o wide                          # Get a list of all services and their external IPs
   ```

2. **Network Policies(netpol):**

  A Network Policy is a Kubernetes object that defines how groups of Pods are allowed to communicate with each other and with external endpoints. By default, all traffic between Pods in a namespace is allowed. Network Policies provide a way to enforce fine-grained firewall rules at the Pod level.

  Network Policies are declarative, meaning they are defined in YAML manifests. You must have a Container Network Interface (CNI) plugin (like Calico or Cilium) that supports Network Policies in your cluster for them to work. There are no direct kubectl create networkpolicy commands. Instead, you create the YAML file and apply it.

   ```sh
    kubectl apply -f backend-policy.yaml                        # Apply the Network Policy manifest to the cluster
    kubectl get networkpolicy                                   # Get a list of all network policies
    kubectl describe networkpolicy backend-policy               # Get detailed information about a specific network policy
   ```

---

## 🔐 ConfigMaps & Secrets 

  ConfigMaps and Secrets are Kubernetes objects used to decouple configuration data from application code. This makes your applications portable and easier to manage across different environments.

  1. **ConfigMaps:**
  Designed for non-sensitive, plaintext configuration data. This could include database hostnames, environment flags, or logging levels.

  ```sh
    # Create a ConfigMap from a literal key-value pair
    kubectl create configmap my-config --from-literal=database.host=db.example.com --from-literal=log.level=info

    # Create a ConfigMap from a single file
    kubectl create configmap my-config-file --from-file=app.properties

    # Create a ConfigMap from all files in a directory
    kubectl create configmap my-config-dir --from-file=config/

    # Get information of configmaps 
    kubectl get configmap                                 # Get a list of all ConfigMaps
    kubectl describe configmap my-config                  # Get detailed information about a specific ConfigMap
    kubectl get configmap my-config -o yaml               # View the contents of the ConfigMap
   ```

  2. **Secrets:**
  Designed for sensitive data, such as passwords, API keys, and tokens. Secrets are base64-encoded, which provides a layer of obfuscation, but they are not encrypted by default. For strong security, you should use an external secrets management solution.

  ```sh
    # Create a generic Secret from literal key-value pairs
    kubectl create secret generic my-secret --from-literal=api_key=my-secure-key --from-literal=db_password=my-strong-password

    # Create a Secret from a file
    kubectl create secret generic my-secret-file --from-file=db_credentials.txt

    # Create a Secret for a Docker registry
    kubectl create secret docker-registry my-registry-secret --docker-server=registry.example.com --docker-username=user --docker-password=pass --docker-email=user@example.com

    # Get information of secrets
    kubectl get secret                                    # Get a list of all Secrets
    kubectl describe secret my-secret                     # Get detailed information about a specific Secret
    kubectl get secret my-secret -o yaml                  # View the base64-encoded contents of the Secret (not recommended)
  ```

---

## 📚 Reference

- [Kubernetes Documentation](https://kubernetes.io/docs/)