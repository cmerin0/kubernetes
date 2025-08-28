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

1. **ConfigMaps(cm):**
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

## 🔐 Volumes

  Unlike containers, Kubernetes Pods are ephemeral. This means that when a Pod is restarted or deleted, any data stored within its filesystem is lost. Volumes solve this problem by providing a way to share data between containers in a Pod and persist data beyond the Pod's lifecycle.

  ### Persistent Volumes(PV):
  Persistent Volume as a specific, pre-provisioned piece of physical storage. like an actual hard drive or SSD. It's the concrete resource in your cluster that is ready to be used. In this method, a cluster administrator manually creates a PV that represents a specific piece of physical storage. A user then creates a Persistent Volume Claim (PVC) to request storage, and Kubernetes tries to find an existing PV that matches the claim's requirements (like size and access mode). This is a manual process and can be difficult to scale.

  ### Common Volume Types
  - emptyDir: An empty directory is created when the Pod is first assigned to a node. It is mounted to the container(s) and provides temporary storage. The data is lost when the Pod is removed from the node.

  - hostPath: Mounts a file or directory from the host node's filesystem into a Pod. This is useful for system-level data but is not recommended for production due to security risks and lack of portability.

  - PersistentVolumeClaim (PVC): This is the most common and recommended way to manage storage in a Kubernetes cluster. It abstracts the underlying storage infrastructure, allowing developers to request storage without knowing the specific details of how it's provisioned.

  ```sh
    # This will create a PVC with 2Gi storage and the default storage class.
    kubectl create pvc my-pvc --storage=2Gi
  ```
  ### StorageClasses(sc)
  A StorageClass provides a way for administrators to describe the "classes" of storage they offer. It defines the provisioner, parameters, and reclaim policy for dynamically provisioned volumes.  This is the more modern and recommended approach. Instead of creating PVs manually, an administrator creates one or more Storage Classes. 

  - When a user creates a PersistentVolumeClaim (PVC) without a specific storageClassName, it will bind to a PV that already exists with no storageClassName, or to the default StorageClass.

  - When a user creates a PVC and specifies a StorageClass, Kubernetes uses the information in that StorageClass to dynamically provision a new PV that matches the PVC's request. This automates the PV creation process, so a cluster administrator doesn't have to manually create a PV for every request.

---

## 🔐 Jobs and CronJobs(cj)

  While Pods are designed for long-running services, Jobs and CronJobs are used to manage finite, one-off, or scheduled tasks

1. **Jobs**
  A Job is a controller that ensures a specified number of Pods run to completion successfully. Once the task is finished, the Pod terminates and does not restart.

  - Running a batch processing task, like an ETL (Extract, Transform, Load) process.
  - Performing a one-time database migration.
  - Executing a script to generate a report.

  ```sh
    # Create a Job from an image (will run the container\'s default command)
    kubectl create job my-one-time-job --image=my-image:latest
  ```

2. **CronJobs(cj)**
  A CronJob creates a Job on a recurring schedule, similar to the cron utility in Linux.

  - Running a nightly database backup.
  - Sending a weekly email newsletter.
  - Periodically checking the health of an external service.

  ```sh
    # This will create a CronJob that runs 'echo "Hello, world!"' every minute.
    kubectl create cronjob my-scheduled-job --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c "echo Hello from the scheduled job!"
  ```
---

## 🔐 About pods: Resources, Probes and Security Context

1. **Resource Limits and Request:**

Resource management is critical for ensuring application stability and efficient cluster utilization. Requests and Limits are used to define the minimum and maximum CPU and memory resources a container can use.

- resources.requests: The minimum amount of CPU or memory that Kubernetes will guarantee for a container. The scheduler uses this value to decide which node to place the Pod on.

- resources.limits: The maximum amount of CPU or memory a container is allowed to use. If a container exceeds its memory limit, the container will be terminated. If a container exceeds its CPU limit, it will be throttled.

2. **Probes and Health Checks:**

Kubernetes uses probes to determine the health of a container. There are three types of probes:

- livenessProbe: Checks if the container is running and healthy. If the probe fails, Kubernetes will restart the container.
- readinessProbe: Checks if the container is ready to serve traffic. If the probe fails, Kubernetes will stop sending traffic to the Pod's Service.
- startupProbe: Checks if the application within the container has started successfully. This is useful for applications that have a long startup time. Once the startup probe succeeds, the liveness and readiness probes begin their checks.

### Probe Types

- httpGet: Performs an HTTP GET request to a specified path on a given port.
- tcpSocket: Checks if a TCP socket can be opened on a specific port.
- exec: Executes a command inside the container and checks the exit code (0 for success).

### Other options for Probes

- initialDelaySeconds: The number of seconds to wait after a container has started before the first probe is performed. This gives the application time to initialize and start listening for requests. For example, setting initialDelaySeconds: 5 means Kubernetes will wait 5 seconds before making the first health check.

- periodSeconds: How often (in seconds) the probe should be performed. The default value is 10 seconds. For example, a periodSeconds: 10 means the probe will run every 10 seconds.

- failureThreshold: The number of consecutive times a probe must fail before Kubernetes considers the container's state to be unhealthy. If a probe fails a number of times greater than or equal to this threshold, the container will be restarted (for livenessProbe) or taken out of service (for readinessProbe). The default value is 3.

3. **Security Context:**

A Security Context defines privilege and access control settings for a Pod or a Container. This helps to secure your application by limiting what it can do inside a cluster.

- Pod-level Security Context: Applies the settings to all containers in the Pod.
- Container-level Security Context: Applies settings only to that specific container. Container-level settings override Pod-level settings.

### Common Settings

* runAsUser: The UID to run the container's entrypoint as.
* runAsGroup: The GID to run the container's entrypoint as.
* allowPrivilegeEscalation: Controls whether a process can gain more privileges than its parent process.
* privileged: Runs the container in privileged mode, giving it access to all devices on the host. This is generally not recommended.
* capabilities: Adds or drops Linux capabilities (e.g., CAP_NET_BIND_SERVICE).

---
## 🔐 Taints & Tolerations and Node Affinity

1. **Taints & Tolerations**

Taints are applied to nodes and repel a Pod from being scheduled on that node unless the Pod has a matching Toleration. Think of a taint as a "no entry" sign for most Pods. You would use taints to designate specific nodes for specific workloads, like isolating sensitive data processing Pods or dedicating high-performance nodes to a particular application. A taint has three parts:

- Key: A name, like dedicated-team.
- Value: An optional value, like data-processing.
- Effect: Defines what happens if a Pod doesn't tolerate the taint.
    1. NoSchedule: Pods without a matching toleration will not be scheduled on the node. Existing Pods on the node are not affected. This is the most common effect.
    2. PreferNoSchedule: The scheduler will try to avoid placing Pods on the node, but will do so if no other suitable nodes are available.
    3. NoExecute: Pods without a matching toleration will be evicted from the node immediately, and new Pods will not be scheduled on it.

Tolerations are applied to Pods and allow them to be scheduled on nodes with a matching taint. The toleration must match the key, value, and effect of the taint.

```sh
    # Taint a node called 'node1'
    kubectl taint nodes node1 dedicated-team=data-processing:NoSchedule

    # To remove the taint from the node
    kubectl taint nodes node1 dedicated-team=data-processing:NoSchedule-

    # You can also add a taint without a value. This is useful for simple flags.
    # This marks a node as having special hardware, repelling non-tolerant pods.
    kubectl taint nodes node2 special-hardware:NoSchedule

    # The 'NoExecute' effect will evict existing pods that don't tolerate the taint.
    # This is useful for nodes that have failed a health check or are undergoing maintenance.
    kubectl taint nodes node3 disk-pressure:NoExecute

    # To check for taints on a node, you can use `kubectl describe`.
    kubectl describe node node1 | grep Taints
```

2. **Node Affinity**

Node Affinity is a property applied to Pods that attracts them to a set of nodes. While taints repel Pods, node affinity attracts them. It is a more flexible and expressive version of nodeSelector and is used to define rules for Pod scheduling. There are two types of node affinity:

- requiredDuringSchedulingIgnoredDuringExecution: This is a hard rule. The scheduler will only place the Pod on a node that meets the defined criteria. If no such node exists, the Pod will remain in a Pending state.

- preferredDuringSchedulingIgnoredDuringExecution: This is a soft rule. The scheduler will try to find a node that meets the criteria, but if it can't, it will still schedule the Pod on a node that doesn't match the rule. This is useful for optimizing Pod placement without preventing them from being scheduled.

Note on IgnoredDuringExecution: This part of the name means that if a node's labels change after the Pod is already running on it, the Pod will not be evicted. The Pod stays where it is.

---

## 📚 Reference

- [Kubernetes Documentation](https://kubernetes.io/docs/)