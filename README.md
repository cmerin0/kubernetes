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

## 📚 Reference

- [Kubernetes Documentation](https://kubernetes.io/docs/)