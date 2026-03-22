When OpenAI tested with 1 million node cluster with kubernetes. I wondered if that experiment can be replicated. Just for the large numbers.
Interesting knowledge in that post is not the numbers, but finding out where the actual bottleneck is. And interestingly even on my single machine with just 2CPUs and 4GB RAM allocated to a minikube, 
I can run a 1000 node mega cluster.

## Part 1: Minikube Setup

### Prerequisites

- macOS (Apple Silicon or Intel)
- Docker Desktop installed and running
- Homebrew installed

### Install Dependencies

```bash
brew install minikube kubectl jq
```

Verify versions:

```bash
minikube version   # expect v1.38.1+
kubectl version --client
```

### Start Minikube with Sufficient Resources

The default minikube profile (2 CPUs / 2 GB RAM) is too small for 1000 KWOK nodes. Use these settings to give the control plane room to breathe:

```bash
minikube start \
  --driver=docker \
  --cpus=6 \
  --memory=8192 \
  --disk-size=50g \
  --kubernetes-version=v1.35.1
```

| Flag | Value | Reason |
|------|-------|--------|
| `--driver=docker` | docker | Uses Docker Desktop — no VM overhead on macOS |
| `--cpus` | 6 | API server + etcd + KWOK controller need headroom |
| `--memory` | 8192 MB | etcd stores all 1000 node objects; 8 GB is comfortable |
| `--disk-size` | 50 GB | Default 20 GB can fill up with container images |
| `--kubernetes-version` | v1.35.1 | Pin to a known-good version |

Wait for minikube to finish. Expected output at the end:

```
Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### Verify the Cluster

```bash
kubectl cluster-info
kubectl get nodes
```

Expected output: one node named `minikube` in `Ready` state with role `control-plane`.

### (Optional) Persist Resource Settings

To avoid passing flags every time you recreate the cluster:

```bash
minikube config set driver docker
minikube config set cpus 6
minikube config set memory 8192
```

Next time you can just run `minikube start`.

### Scaling Resources Later

If you want to push beyond 5000 nodes:

```bash
minikube stop
minikube config set memory 16384
minikube config set cpus 10
minikube start
```

---

## Part 2: KWOK Setup

### Step 1: Install KWOK CLI

```bash
brew install kwok
```

Verify installation:

```bash
kwok --version
```


```bash
➜  minikube kubectl get pods -o wide -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE     IP             NODE       NOMINATED NODE   READINESS GATES
kube-system   coredns-7d764666f9-klp9z           1/1     Running   1 (13m ago)   3d13h   10.244.0.3     minikube   <none>           <none>
kube-system   etcd-minikube                      1/1     Running   1 (13m ago)   3d13h   192.168.49.2   minikube   <none>           <none>
kube-system   kube-apiserver-minikube            1/1     Running   1 (13m ago)   3d13h   192.168.49.2   minikube   <none>           <none>
kube-system   kube-controller-manager-minikube   1/1     Running   1 (13m ago)   3d13h   192.168.49.2   minikube   <none>           <none>
kube-system   kube-proxy-9gpss                   1/1     Running   1 (13m ago)   3d13h   192.168.49.2   minikube   <none>           <none>
kube-system   kube-scheduler-minikube            1/1     Running   1 (13m ago)   3d13h   192.168.49.2   minikube   <none>           <none>
kube-system   storage-provisioner                1/1     Running   3 (13m ago)   3d13h   192.168.49.2   minikube   <none>           <none>
➜  minikube brew install kwok
==> Auto-updating Homebrew...
Adjust how often this is run with `$HOMEBREW_AUTO_UPDATE_SECS` or disable with
`$HOMEBREW_NO_AUTO_UPDATE=1`. Hide these hints with `$HOMEBREW_NO_ENV_HINTS=1` (see `man brew`).
==> Auto-updated Homebrew!
Updated 2 taps (homebrew/core and homebrew/cask).
==> New Formulae
apache-arrow-adbc: Cross-language, Arrow-native database access
cline: AI-powered coding agent for complex work
ctx7: Manage AI coding skills and documentation context
cyan: iOS app injector and modifier
lief: Library to Instrument Executable Formats
portless: Replace port numbers with stable, named local URLs for humans and agents
wmbusmeters: Read wired or wireless mbus protocol to acquire utility meter readings
==> New Casks
dbeaverteam: Universal database tool and SQL client
dbvr: Lightweight CLI tool for running database operations
font-selawik

You have 39 outdated formulae and 3 outdated casks installed.

==> Fetching downloads for: kwok
✔︎ Bottle Manifest kwok (0.7.0)                                                                                                                                                                                                                                                                                                          Downloaded    7.5KB/  7.5KB
✔︎ Bottle kwok (0.7.0)                                                                                                                                                                                                                                                                                                                   Downloaded   69.6MB/ 69.6MB
==> Pouring kwok--0.7.0.arm64_tahoe.bottle.1.tar.gz
🍺  /opt/homebrew/Cellar/kwok/0.7.0: 15 files, 166.8MB
==> Running `brew cleanup kwok`...
Disable this behaviour by setting `HOMEBREW_NO_INSTALL_CLEANUP=1`.
Hide these hints with `HOMEBREW_NO_ENV_HINTS=1` (see `man brew`).
==> Caveats
zsh completions have been installed to:
  /opt/homebrew/share/zsh/site-functions
➜  minikube kwok --version

kwok version v0.7.0 go1.25.5 (darwin/arm64)

```


---

### Step 2: Deploy KWOK Controller into Minikube

KWOK runs as a controller inside your cluster. It intercepts node heartbeats and keeps fake nodes in `Ready` state without real kubelets.

```bash
KWOK_REPO=kubernetes-sigs/kwok
KWOK_LATEST_RELEASE=$(curl -s "https://api.github.com/repos/${KWOK_REPO}/releases/latest" | jq -r '.tag_name')

echo "Installing KWOK version: ${KWOK_LATEST_RELEASE}"

# Deploy the KWOK controller and CRDs
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/kwok.yaml"

# Deploy stage configuration (defines fake node/pod lifecycle behavior)
kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/stage-fast.yaml"
```

Wait for the controller to be ready:

```bash
kubectl -n kube-system rollout status deployment/kwok-controller --timeout=60s
```

Verify it is running:

```bash
kubectl -n kube-system get pods -l app=kwok-controller
```


```bash
➜  minikube kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/kwok.yaml"
customresourcedefinition.apiextensions.k8s.io/attaches.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterattaches.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterexecs.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterlogs.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterportforwards.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/clusterresourceusages.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/execs.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/logs.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/metrics.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/portforwards.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/resourceusages.kwok.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/stages.kwok.x-k8s.io created
serviceaccount/kwok-controller created
clusterrole.rbac.authorization.k8s.io/kwok-controller created
clusterrolebinding.rbac.authorization.k8s.io/kwok-controller created
configmap/kwok created
service/kwok-controller created
deployment.apps/kwok-controller created
flowschema.flowcontrol.apiserver.k8s.io/kwok-controller created
➜  minikube kubectl apply -f "https://github.com/${KWOK_REPO}/releases/download/${KWOK_LATEST_RELEASE}/stage-fast.yaml"
stage.kwok.x-k8s.io/node-heartbeat-with-lease created
stage.kwok.x-k8s.io/node-initialize created
stage.kwok.x-k8s.io/pod-complete created
stage.kwok.x-k8s.io/pod-delete created
stage.kwok.x-k8s.io/pod-ready created
➜  minikube kubectl -n kube-system rollout status deployment/kwok-controller --timeout=60s
deployment "kwok-controller" successfully rolled out
➜  minikube kubectl -n kube-system get pods -l app=kwok-controller
NAME                               READY   STATUS    RESTARTS   AGE
kwok-controller-5cf8c7c44b-lft7r   1/1     Running   0          23s
```

---

### Step 3: Create 1000 Fake Nodes

The script below generates and applies 1000 node manifests in a single pass. Each node:

- Is named `fake-node-0001` through `fake-node-1000`
- Has the label `type=kwok` for easy selection
- Carries the KWOK taint so real workloads do not accidentally land on them

```bash
for i in $(seq 1 1000); do
cat <<EOF
apiVersion: v1
kind: Node
metadata:
  name: fake-node-$(printf "%04d" $i)
  annotations:
    kwok.x-k8s.io/node: fake
  labels:
    type: kwok
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    kubernetes.io/arch: amd64
    kubernetes.io/hostname: fake-node-$(printf "%04d" $i)
    kubernetes.io/os: linux
    node.kubernetes.io/exclude-from-external-load-balancers: ""
spec:
  taints:
  - effect: NoSchedule
    key: kwok.x-k8s.io/node
    value: fake
---
EOF
done | kubectl apply -f -
```

This will take 30–90 seconds depending on API server load.

---

### Step 4: Verify the Nodes

Check total node count (should be 1001: 1000 fake + 1 real minikube node):

```bash
kubectl get nodes | wc -l
```

Check only the fake nodes:

```bash
kubectl get nodes --selector=type=kwok
```

Confirm they are all in `Ready` state:

```bash
kubectl get nodes --selector=type=kwok --no-headers | awk '{print $2}' | sort | uniq -c
```

Expected output:
```
1000 Ready
```

```
fake-node-0999   Ready    <none>   20s   kwok-v0.7.0
fake-node-1000   Ready    <none>   20s   kwok-v0.7.0
➜  minikube kubectl get nodes --selector=type=kwok --no-headers | awk '{print $2}' | sort | uniq -c
1000 Ready
```

---

### Step 5: Schedule Test Pods on Fake Nodes (Optional)

Fake nodes have a `NoSchedule` taint, so you must add a toleration to any pod you want scheduled there. Use this example deployment to verify scheduling works:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fake-workload
spec:
  replicas: 100
  selector:
    matchLabels:
      app: fake-workload
  template:
    metadata:
      labels:
        app: fake-workload
    spec:
      tolerations:
      - key: kwok.x-k8s.io/node
        operator: Exists
        effect: NoSchedule
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: type
                operator: In
                values:
                - kwok
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: 10m
            memory: 16Mi
```

Apply and check:

```bash
kubectl apply -f fake-workload.yaml
kubectl get pods -l app=fake-workload -o wide
```


```
➜  minikube kubectl get pods -l app=fake-workload -o wide
NAME                            READY   STATUS    RESTARTS   AGE   IP             NODE             NOMINATED NODE   READINESS GATES
fake-workload-bccff8786-25w6s   1/1     Running   0          6s    10.0.0.15      fake-node-0463   <none>           <none>
fake-workload-bccff8786-2ljg6   1/1     Running   0          3s    10.0.0.58      fake-node-0625   <none>           <none>
fake-workload-bccff8786-2x2qb   1/1     Running   0          2s    10.0.0.69      fake-node-0752   <none>           <none>
fake-workload-bccff8786-44rgz   1/1     Running   0          2s    10.0.0.67      fake-node-0845   <none>           <none>
fake-workload-bccff8786-49bjk   1/1     Running   0          4s    10.0.0.37      fake-node-0842   <none>           <none>
fake-workload-bccff8786-4c9gz   1/1     Running   0          2s    10.244.180.0   fake-node-0175   <none>           <none>
fake-workload-bccff8786-4mb9k   1/1     Running   0          5s    10.0.0.26      fake-node-0302   <none>           <none>
fake-workload-bccff8786-5jrxn   1/1     Running   0          5s    10.244.102.0   fake-node-0097   <none>           <none>
```
---


**Kubernetes is a control loop engine that happens to manage containers.** The API server, etcd, the scheduler, every controller — they're all operating on abstract objects. The fact that those objects usually represent real machines with real processes is almost incidental. Strip away the kubelets and container runtimes, and what you're left with is still a fully functional distributed state machine. Nodes are rows in a database. Pods are rows in a database. The controllers are just programs that read rows, make decisions, and write rows back.


![](Pasted%20image%2020260321215046.png)

![](Pasted%20image%2020260321215103.png)