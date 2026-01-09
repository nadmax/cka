# CKA Cheatsheet

This cheatsheet is based on [CKA scenarios on Killercoda](https://killercoda.com/cka).  
Feel free to add uncovered topics for the CKA.

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

- `kubectl` Cheat Sheet: `/docs/reference/kubectl/cheatsheet/`
- Pod spec examples: `/docs/concepts/workloads/pods/`
- NetworkPolicy examples: `/docs/concepts/services-networking/network-policies/`
- RBAC examples: `/docs/reference/access-authn-authz/rbac/`
- Storage examples: `/docs/concepts/storage/`

## Environment Setup

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

## Control Plane Components

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

## Troubleshooting Applications

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

## ConfigMap Usage

### Mounting ConfigMap as Volume

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

### Using ConfigMap as Environment Variables

```yml
env:
- name: <ENV_VAR_NAME>
  valueFrom:
    configMapKeyRef:
      name: <configmap-name>
      key: <key-name>
```

### Verify ConfigMap Data

```sh
controlplane:~$ k exec <pod-name> -- env | grep <ENV_VAR>
controlplane:~$ k exec <pod-name> -- cat <mount-path>/<key-name>
```

## Services and Ingress

### Create ClusterIP Service

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
  ingressClassName: nginx
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

## Resource Management

### Find Pods by QoS Class

```sh
# Find BestEffort pods (first to be evicted)
controlplane:~$ k get pods -A -o yaml | grep -i besteffort

# Find Burstable or Guaranteed
controlplane:~$ k get pods -A -o yaml | grep -i "burstable\|guaranteed"
```

## Network Policies

### Egress Policy Template

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

### Ingress Policy Template

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

## RBAC (Role-Based Access Control)

### RBAC Combinations

- **Role + RoleBinding**: Available and applied in single namespace
- **ClusterRole + ClusterRoleBinding**: Available and applied cluster-wide
- **ClusterRole + RoleBinding**: Available cluster-wide, applied in single namespace
- **Role + ClusterRoleBinding**: NOT POSSIBLE

### Create ServiceAccount and Roles

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

### Verify Permissions

```sh
controlplane:~$ k auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>
```

## Pod Scheduling

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

## DaemonSet with HostPath

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

## Cluster Management

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

## Cluster Upgrade

### Upgrade Controlplane

```sh
# Check for new version available
controlplane:~$ kubeadm upgrade plan

# Apply upgrade
controlplane:~$ kubeadm upgrade apply <version>

# Upgrade kubelet and kubectl
controlplane:~$ apt install -y kubelet=<version> kubectl=<version>
```

### Upgrade Worker Node

```sh
controlplane:~$ ssh <worker-node-name>

# Upgrade kubeadm first with the upgraded version
<worker-node-name>:~$ apt install kubeadm=<version>

# Upgrade worker node
<worker-node-name>:~$ kubeadm upgrade node

# Upgrade kubelet and kubectl
<worker-node-name>:~$ apt install -y kubelet=<version> kubectl=<version>
```

## Certificate Management

### Check Certificate Expiration

```sh
controlplane:~$ kubeadm certs check-expiration
```

### Renew Certificates

```sh
# Renew all certificates
controlplane:~$ kubeadm certs renew all

# Renew specific certificate
controlplane:~$ kubeadm certs renew <certificate-name>
```

## Static Pods

Static pods are managed by kubelet directly from manifest files.

### Move Static Pod Between Nodes

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
