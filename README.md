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

## 🔐 Pods and Multi-Container Pods 

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

## 📚 Reference

- [Kubernetes Documentation](https://kubernetes.io/docs/)