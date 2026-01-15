# CKA Cheatsheet

This cheatsheet is based on [CKA scenarios on Killercoda](https://killercoda.com/cka).

## Important Note About the Exam

**The official Kubernetes documentation is available during the exam at [kubernetes.io/docs](https://kubernetes.io/docs)**

- All answers can be found in the documentation
- Practice navigating the docs efficiently before the exam
- Familiarize yourself with the documentation structure:
  - **Concepts**: Understanding Kubernetes components
  - **Tasks**: Step-by-step guides (most useful during exam)
  - **Reference**: API specs, kubectl commands, and component flags
- Use the search function effectively
- Bookmark commonly used pages if the exam environment allows
- Know where to find YAML examples for common resources

**Key documentation sections to know:**

- [`kubectl` Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet)
- [Pod spec examples](https://kubernetes.io/docs/concepts/workloads/pods)
- [NetworkPolicy examples](https://kubernetes.io/docs/concepts/services-networking/network-policies)
- [RBAC examples](https://kubernetes.io/docs/reference/access-authn-authz/rbac)
- [Storage examples](https://kubernetes.io/docs/concepts/storage)

---

## 1. Environment Setup

### Shortcuts

Since the exam takes 2 hours, I recommend only using the following shortcuts.

```sh
controlplane:~$ alias k=kubectl
controlplane:~$ export do="--dry-run=client -o yaml" # Variable for generating YAML without creating resources
```

### Vim (Optional)

If you would like to customize vim, here's what I recommend.

```sh
controlplane:~$ echo "set et ts=2 sw=2 nu" >> ~/.vimrc
```

This sets: `expandtab`, `tabstop=2`, `shiftwidth=2` and `line numbers`

---

## 2. Cluster Architecture and Management

### Initialize Cluster

```sh
controlplane:~$ kubeadm init \
  --kubernetes-version <version> \
  --pod-network-cidr <cidr>
```

### Join Node to Cluster

```sh
controlplane:~$ kubeadm token create --print-join-command
<worker-node-name>:~$ # copy/paste the output of 'kubeadm token create' command
```

### Cluster Upgrade

#### Upgrade Controlplane

```sh
# Check for new version available
controlplane:~$ kubeadm upgrade plan

# Apply upgrade
controlplane:~$ kubeadm upgrade apply <version>

# Upgrade kubelet and kubectl
controlplane:~$ apt install -y kubelet=<version> kubectl=<version>
```

#### Upgrade Worker Node

```sh
controlplane:~$ ssh <worker-node-name>

# Upgrade kubeadm first with the upgraded version
<worker-node-name>:~$ apt install kubeadm=<version>

# Upgrade worker node
<worker-node-name>:~$ kubeadm upgrade node

# Upgrade kubelet and kubectl
<worker-node-name>:~$ apt install -y kubelet=<version> kubectl=<version>
```

### Certificate Management

#### Check Certificate Expiration

```sh
controlplane:~$ kubeadm certs check-expiration
```

#### Renew Certificates

```sh
# Renew all certificates
controlplane:~$ kubeadm certs renew all

# Renew specific certificate
controlplane:~$ kubeadm certs renew <certificate-name>
```

---

## 3. Control Plane Components

### API Server

```sh
controlplane:~$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Check for invalid options or misconfigurations in the static pod manifest.

### Kube Controller Manager

```sh
controlplane:~$ vim /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Verify configuration flags and ensure all options are valid.

### Kubelet

```sh
controlplane:~$ vim /var/lib/kubelet/config.yaml
# or check environment file
controlplane:~$ vim /var/lib/kubeadm-flags.env
```

Review kubelet configuration and check for invalid flags.

---

## 4. Workload Management

### Rollback and Rollout

```sh
controlplane:~$ k set image deployment <deploy-name> <image>=<image> <image>=<image>:<tag> # Change the image of our deployment
controlplane:~$ k rollout history deploy apache # View rollout history
controlplane:~$ k rollout undo deploy apache # Roll back to a previous version
controlplane:~$ k rollout status deploy apache # View rollout status
```

### Rollout Strategy

```yml
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: source-ip-app
  strategy:
    ## rollingUpdate:
      ## maxSurge: 25%
        ## maxUnavailable: 25%
    ## type: RollingUpdate
    type: Recreate
```

```sh
controlplane:~$ k get deploy <deploy-name> -oyaml | grep strategy -A3 # Verify that changes were successful
```

### Static Pods

Static pods are managed by kubelet directly from manifest files.

#### Move Static Pod Between Nodes

```sh
# Copy from source node
controlplane:~$ scp <source-node>:/etc/kubernetes/manifests/<pod-manifest> .

# Remove from source (stops the pod)
controlplane:~$ ssh <source-node> -- rm /etc/kubernetes/manifests/<pod-manifest>

# Modify as needed (e.g., change name)
controlplane:~$ vim <pod-manifest>

# Move to Kubernetes manifests directory
controlplane:~$ mv <pod-manifest> /etc/kubernetes/manifests/
```

### DaemonSet with HostPath

```yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: <daemonset-name>
  namespace: <namespace>
spec:
  selector:
    matchLabels:
      <key>: <value>
  template:
    metadata:
      labels:
        <key>: <value>
    spec:
      containers:
      - name: <container-name>
        image: <image>
        command:
        - sh
        - -c
        - '<command> && sleep infinity'
        volumeMounts:
        - name: <volume-name>
          mountPath: <container-path>
      volumes:
      - name: <volume-name>
        hostPath:
          path: <host-path>
```

---

## 5. Pod Scheduling

### NodeName

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
  namespace: <namespace>
spec:
  containers:
  - image: <image>
    name: <container-name>
  nodeName: <node-name>
```

### Taints and Toleration

```sh
controlplane:~$ k taint nodes <node-name> key1=value1:NoSchedule
```

This means that no pod will be able to schedule onto the node unless it has a matching toleration.

```sh
controlplane:~$ k taint no <node-name> key1=value1:NoSchedule- # untaint node
```

Toleration on a pod:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  tolerations:
  - key: "key1"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - image: <image-name>
    name: <container-name>
  nodeSelector:
    kubernetes.io/hostname: <node-name>
```

### Cordon and Drain

```sh
controlplane:~$ k cordon <node-name> # Mark node as unschedulable
controlplane:~$ k drain <node-name> --ignore-daemonsets # Evict all pods currently running on node (without --ignore-daemonsets, an error occurred)
controlplane:~$ k uncordon <node-name> # Enable scheduling
```

### Priority

```sh
# Find pods by priority
controlplane:~$ k get pods -n <namespace> -o yaml | grep -i priority -B 20
```

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  priority: <priority-value>
  priorityClassName: <priority-class-name>
  containers:
  - name: <container-name>
    image: <image>
```

### Pod Affinity

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: <label-key>
            operator: In
            values:
            - <label-value>
        topologyKey: kubernetes.io/hostname
  containers:
  - image: <image>
    name: <container-name>
```

### Pod Anti-Affinity

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: <label-key>
            operator: In
            values:
            - <label-value>
        topologyKey: kubernetes.io/hostname
  containers:
  - image: <image>
    name: <container-name>
```

---

## 6. Configuration

### ConfigMap Usage

#### From Literal

```sh
controlplane:~$ k create configmap <name> --from-literal=<ENV>=<VALUE>
```

#### Mounting ConfigMap as Volume

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
  namespace: <namespace>
spec:
  volumes:
  - name: <volume-name>
    configMap:
      name: <configmap-name>
  containers:
  - image: <image>
    name: <container-name>
    volumeMounts:
    - name: <volume-name>
      mountPath: <mount-path>
```

#### Using ConfigMap as Environment Variables

```yml
env:
- name: <ENV_VAR_NAME>
  valueFrom:
    configMapKeyRef:
      name: <configmap-name>
      key: <key-name>
```

#### Verify ConfigMap Data

```sh
controlplane:~$ k exec <pod-name> -- env | grep <ENV_VAR>
controlplane:~$ k exec <pod-name> -- cat <mount-path>/<key-name>
```

### Secrets

#### From Literal

```sh
controlplane:~$ k create secret <secret-type> <name> --from-literal=<ENV>=<VALUE>
```

#### Service Account Token

```yml
apiVersion: v1
kind: Secret
metadata:
    name: <secret-name>
    namespace: <namespace-name>
    annotations:
        kubernetes.io/service-account.name: "<sa-name>"
type: kubernetes.io/service-account-token
```

---

## 7. Networking

### Services

#### Create ClusterIP Service

```sh
controlplane:~$ k create service clusterip <service-name> --tcp=<port>:<target-port> -n <namespace>
```

### Ingress Resource

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <ingress-name>
  namespace: <namespace>
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx # Uses nginx implementation
  rules:
  - host: "<hostname>"
    http:
      paths:
      - path: /<path>
        pathType: Prefix
        backend:
          service:
            name: <service-name>
            port:
              number: <port>
```

### Gateway and HTTPRoute

```yml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: <gateway-name>
  namespace: <name>
spec:
  gatewayClassName: nginx # Uses nginx implementation
  listeners:
  - protocol: HTTP
    port: 80
    name: <name>
```

```yml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path: # Uses a path-based routing
        type: PathPrefix
        value: /
    backendRefs: # References web service
    - name: web
      port: 80
```

### Network Policies

#### Egress Policy Template

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <policy-name>
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      <key>: <value>  # {} for all pods
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          <label-key>: <label-value>
```

#### Ingress Policy Template

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <policy-name>
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      <key>: <value>  # {} for all pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          <label-key>: <label-value>
    - podSelector:
        matchLabels:
          <label-key>: <label-value>
```

---

## 8. Security

### RBAC (Role-Based Access Control)

#### RBAC Combinations

- **Role + RoleBinding**: Available and applied in single namespace
- **ClusterRole + ClusterRoleBinding**: Available and applied cluster-wide
- **ClusterRole + RoleBinding**: Available cluster-wide, applied in single namespace
- **Role + ClusterRoleBinding**: NOT POSSIBLE

#### Create ServiceAccount and Roles

```sh
# Create ServiceAccount
controlplane:~$ k create serviceaccount <sa-name> -n <namespace>

# Create Role
controlplane:~$ k create role <role-name> \
  --verb=<verbs> \
  --resource=<resources> \
  -n <namespace>

# Create RoleBinding
controlplane:~$ k create rolebinding <binding-name> \
  --role=<role-name> \
  --serviceaccount=<namespace>:<sa-name> \
  -n <namespace>

# Create ClusterRole
controlplane:~$ k create clusterrole <role-name> \
  --verb=<verbs> \
  --resource=<resources>

# Create ClusterRoleBinding
controlplane:~$ k create clusterrolebinding <binding-name> \
  --clusterrole=<role-name> \
  --serviceaccount=<namespace>:<sa-name>
```

#### Verify Permissions

```sh
controlplane:~$ k auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>
```

---

## 9. Storage

### StorageClass

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: <storage-class-name>
provisioner: <provisioner-name>
reclaimPolicy: Retain
```

### PV and PVC

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <pv-name>
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: "local-path"
  persistentVolumeReclaimPolicy: Delete # Can be Recycle or Retain too
  hostPath:
    path: "<path>"
```

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <pvc-name>
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
    - ReadWriteOnce 
  storageClassName: "local-path"
  volumeName: <pv-name>
```

Pod with PVC:

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  containers:
    - name: <container-name>
      image: <image-name>
      volumeMounts:
        - mountPath: "<path>"
          name: <volume-name>
  volumes:
    - name: <volume-name>
      persistentVolumeClaim:
        claimName: <pvc-name>
```

### NFS Volume

```yml
apiVersion: v1
kind: Pod
metadata:
  name: <pod-name>
spec:
  containers:
    - name: <container-name>
      image: <image-name>
      command: [ 'sh', '-c', 'while true; do echo "some text" >> /data/test; sleep 3600; done' ]
      volumeMounts:
        - name: <volume-name>
          mountPath: <path>
  volumes:
    - name: <volume-name>
      nfs:
        server: <ip-address>
        path: <path>
        readOnly: no
```

---

## 10. Resource Management and Monitoring

### Find Pods by QoS Class

```sh
# Find BestEffort pods (first to be evicted)
controlplane:~$ k get pods -A -o yaml | grep -i besteffort

# Find Burstable or Guaranteed
controlplane:~$ k get pods -A -o yaml | grep -i "burstable\|guaranteed"
```

### Metrics Server

#### Nodes

```sh
controlplane:~$ k top no # Show metrics for all nodes
controlplane:~$ k top no <node-name> # Show metrics for specific node
```

#### Pods

```sh
controlplane:~$ k top po <name> -- containers # Show metrics for pod and its containers
controlplane:~$ k top po --sort-by=cpu # Sort by pods which are consuming the most CPU
controlplane:~$ k top po --sort-by=memory # Sort by pods which are consuming the most memory
```

---

## 11. Troubleshooting

### Check Pod Logs

```sh
controlplane:~$ k logs <pod-name> -n <namespace>
controlplane:~$ k logs <pod-name> -c <container-name>  # for multi-container pods
```

### Common Issues

- Resource references (ConfigMap, Secret, PVC not found)
- Image pull errors
- Incorrect nodeName specification
- Port conflicts in multi-container pods

---

## 12. Package Management

### Helm

```sh
controlplane:~$ helm version # Verify if Helm is installed and check version
controlplane:~$ helm repo add <repo-name> <url> # Add Helm repository
controlplane:~$ helm repo update # Update repository to fetch the latest chart information
controlplane:~$ helm search repo <repo-name> # Search for available repository charts
controlplane:~$ helm show chart <repo-name>/<chart> # Get detailed information about a chart
controlplane:~$ helm show values <repo-name>/<chart> # View the default values for a chart
controlplane:~$ helm repo list # List the configured repositories
controlplane:~$ helm list -A # List all Helm releases
controlplane:~$ helm install <release-name> <repo-name>/<chart> --namespace <namespace> # Install chart in a namespace
controlplane:~$ helm history <name> -n <namespace> # Get the release history
controlplane:~$ helm get manifest <release-name> -n <namespace> # View generated Kubernetes manifests
controlplane:~$ helm rollback <release-name> <revision-number> -n <namespace> # Rollback to a previous revision
controlplane:~$ helm status <release-name> -n <namespace> # Check release installation status
controlplane:~$ helm get values <release-name> -n <namespace> # Get values for a current release
controlplane:~$ helm get all <release-name> -n <namespace> # Get all information about a release
```

### Kustomize

Example of a basic `kustomization.yaml`

```yml
resources:
- deployment.yaml
- service.yaml
labels:
    env: dev
    team: platform
```

#### Overlays

##### With patchesStrategyMerge

```sh
controlplane:~$ 
.
|-- base
|   |-- deployment.yaml
|   |-- kustomization.yaml
|   `-- service.yaml
`-- overlays
    |-- dev
    |   |-- kustomization.yaml
    |   `-- patch-deployment.yaml
    `-- prod
        |-- kustomization.yaml
        `-- patch-deployment.yaml
```

`overlays/prod/kustomization.yaml`

```yml
resources:
- ../../base
patchesStrategyMerge:
- patch-deployment

namePrefix: -prod
namespace: prod
```

`overlays/prod/patch-deployment.yaml`

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:prod
          env:
            - name: ENV
              value: "production"
```

##### With images

```yml
resources:
- ../../base
images:
- name: nginx
  newTag: "1.23"
```

#### ConfigMapGenerator

```yml
configMapGenerator:
- name: configmap
  files:
  - application.properties
```

#### SecretGenerator

```yml
secretGenerator:
- name: secret
  files:
  - password.txt
```
