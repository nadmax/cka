# CKA Cheatsheet

This cheatsheet is based on [CKA scenarios on Killercoda](https://killercoda.com/cka).  
Feel free to add uncovered topics for the CKA.

## Environment Setup

### Essential Aliases

Add these to your `~/.bashrc` or `~/.bash_profile`:

```sh
# kubectl shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgn='kubectl get nodes'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias ka='kubectl apply -f'
alias kc='kubectl create'

# namespace shortcuts
alias kn='kubectl config set-context --current --namespace'
alias kgns='kubectl get namespaces'

# YAML output
alias kgy='kubectl get -o yaml'
alias kgyp='kubectl get pods -o yaml'

# Watch resources
alias kgpw='kubectl get pods -w'

# Quick dry-run for generating YAML
alias kdr='kubectl run test --image=nginx --dry-run=client -o yaml'
alias kdrd='kubectl create deployment test --image=nginx --dry-run=client -o yaml'

# Enable autocompletion
source <(kubectl completion bash)
complete -F __start_kubectl k
```

Apply changes:

```sh
source ~/.bashrc
```

### Vim Configuration

Create or edit `~/.vimrc`:

```vim
" Enable syntax highlighting
syntax on

" Show line numbers
set number

" Set tab width to 2 spaces (YAML standard)
set tabstop=2
set shiftwidth=2
set expandtab

" Enable auto-indentation
set autoindent
set smartindent

" Show matching brackets
set showmatch

" Enable search highlighting
set hlsearch
set incsearch

" Ignore case in search unless uppercase is used
set ignorecase
set smartcase

" Show cursor position
set ruler

" Enable mouse support
set mouse=a

" Highlight current line
set cursorline

" Better backspace behavior
set backspace=indent,eol,start

" Show command in bottom bar
set showcmd

" Faster updates for better experience
set updatetime=300

" Enable file type detection
filetype plugin indent on

" Paste mode toggle (F2)
set pastetoggle=<F2>

" Quick save with Ctrl+s (add to .bashrc: stty -ixon)
nnoremap <C-s> :w<CR>
inoremap <C-s> <Esc>:w<CR>a

" Quick quit
nnoremap <C-q> :q<CR>
```

### kubectl Quick Reference

```sh
# Set default editor to vim
export KUBE_EDITOR=vim

# Shorter output format
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json

# Imperative commands for speed
kubectl run <pod-name> --image=<image>
kubectl create deployment <name> --image=<image>
kubectl expose deployment <name> --port=<port> --target-port=<target-port>
kubectl scale deployment <name> --replicas=<count>

# Generate YAML without creating resources
kubectl run <pod-name> --image=<image> --dry-run=client -o yaml > pod.yaml
kubectl create deployment <name> --image=<image> --dry-run=client -o yaml > deployment.yaml
```

## Control Plane Components

### API Server

```sh
vim /etc/kubernetes/manifests/kube-apiserver.yaml
```

Check for invalid options or misconfigurations in the static pod manifest.

### Kube Controller Manager

```sh
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
```

Verify configuration flags and ensure all options are valid.

### Kubelet

```sh
vim /var/lib/kubelet/config.yaml
# or check environment file
vim /var/lib/kubeadm.env
```

Review kubelet configuration and check for invalid flags.

## Troubleshooting Applications

### Check Pod Logs

```sh
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -c <container-name>  # for multi-container pods
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

```sh
env:
- name: <ENV_VAR_NAME>
  valueFrom:
    configMapKeyRef:
      name: <configmap-name>
      key: <key-name>
```

### Verify ConfigMap Data

```sh
kubectl exec <pod-name> -- env | grep <ENV_VAR>
kubectl exec <pod-name> -- cat <mount-path>/<key-name>
```

## Services and Ingress

### Create ClusterIP Service

```sh
kubectl create service clusterip <service-name> --tcp=<port>:<target-port> -n <namespace>
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
kubectl get pods -A -o yaml | grep -i besteffort

# Find Burstable or Guaranteed
kubectl get pods -A -o yaml | grep -i "burstable\|guaranteed"
```

## Network Policies

### Egress Policy Template

```sh
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

```sh
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
kubectl create serviceaccount <sa-name> -n <namespace>

# Create Role
kubectl create role <role-name> \
  --verb=<verbs> \
  --resource=<resources> \
  -n <namespace>

# Create RoleBinding
kubectl create rolebinding <binding-name> \
  --role=<role-name> \
  --serviceaccount=<namespace>:<sa-name> \
  -n <namespace>

# Create ClusterRole
kubectl create clusterrole <role-name> \
  --verb=<verbs> \
  --resource=<resources>

# Create ClusterRoleBinding
kubectl create clusterrolebinding <binding-name> \
  --clusterrole=<role-name> \
  --serviceaccount=<namespace>:<sa-name>
```

### Verify Permissions

```sh
kubectl auth can-i <verb> <resource> \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>
```

## Pod Scheduling

### Priority

```sh
# Find pods by priority
kubectl get pods -n <namespace> -o yaml | grep -i priority -B 20
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
kubeadm init \
  --kubernetes-version <version> \
  --pod-network-cidr <cidr> \
  --apiserver-advertise-address <ip>
```

### Generate Join Command

```sh
kubeadm token create --print-join-command
```

### Join Node to Cluster

```sh
# Run on worker node
kubeadm join <control-plane-endpoint> --token <token> --discovery-token-ca-cert-hash <hash>
```

## Cluster Upgrade

### Check Available Versions

```sh
kubeadm upgrade plan
```

### Upgrade Control Plane

```sh
# Upgrade kubeadm first
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=<version>
apt-mark hold kubeadm

# Apply upgrade
kubeadm upgrade apply <version>

# Upgrade kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=<version> kubectl=<version>
apt-mark hold kubelet kubectl

# Restart kubelet
systemctl daemon-reload
systemctl restart kubelet
```

### Upgrade Worker Node

```sh
# Drain node from control plane
kubectl drain <node-name> --ignore-daemonsets

# On worker node
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=<version>
apt-mark hold kubeadm

kubeadm upgrade node

apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=<version> kubectl=<version>
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

# Uncordon from control plane
kubectl uncordon <node-name>
```

## Certificate Management

### Check Certificate Expiration

```sh
kubeadm certs check-expiration
```

### Renew Certificates

```sh
# Renew all certificates
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew <certificate-name>
```

## Static Pods

Static pods are managed by kubelet directly from manifest files.

### Default Location

```sh
/etc/kubernetes/manifests/
```

### Move Static Pod Between Nodes

```sh
# Copy from source node
scp <source-node>:/etc/kubernetes/manifests/<pod-manifest> .

# Remove from source (stops the pod)
ssh <source-node> -- rm /etc/kubernetes/manifests/<pod-manifest>

# Modify as needed (e.g., change name)
vim <pod-manifest>

# Copy to destination
scp <pod-manifest> <dest-node>:/etc/kubernetes/manifests/
```

## Useful kubectl Commands

```sh
# Get resources across all namespaces
kubectl get <resource> -A

# Describe resource for details
kubectl describe <resource> <name> -n <namespace>

# Edit resource directly
kubectl edit <resource> <name> -n <namespace>

# Export resource YAML
kubectl get <resource> <name> -n <namespace> -o yaml > <file.yaml>

# Apply configuration
kubectl apply -f <file.yaml>

# Delete resource
kubectl delete <resource> <name> -n <namespace>

# Force delete pod
kubectl delete pod <pod-name> --force --grace-period=0
```
