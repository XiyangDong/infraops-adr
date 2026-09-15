# Spike: Karmada CR Propagation — Does it Decompose CRs into Native Objects?

**Date:** 2026-09-15  
**Author:** Xiyang Dong  
**JIRA:** AIPCC-32014  
**Status:** Complete — hypothesis refuted

---

## Question

Berto proposed Karmada as a replacement for multi-kueue in our multi-cluster architecture.
His hypothesis: Karmada's Resource Interpreter decomposes CRs like `TrainJob`/`RayJob` into
native Kubernetes objects (Pods, Deployments), so GPU worker clusters would not need ML
operators (training-operator, KubeRay) installed.

**Does Karmada decompose CRs into native objects, or does it propagate the full CR as-is?**

This determines whether GPU worker clusters must have operators installed or can remain generic.

---

## Environment

| Role | Cluster | Notes |
|------|---------|-------|
| Karmada control plane host | rocks (IBM Cloud ROKS, OCP 4.20) | `c114-e.eu-de.containers.cloud.ibm.com:32358` |
| Member cluster | rocks (used as stand-in for infraops) | Joined as `infraops` |

Both clusters are OpenShift 4.20 on IBM Cloud (ROKS). The member cluster was the
rocks cluster itself, registered under the name `infraops`, because infraops credentials
were expired at time of testing. This does not affect the validity of the findings —
what matters is observing the Work object content and what the execution controller
applies to the member cluster.

**Karmada version:** v1.19.0

---

## Setup Steps

These steps install Karmada on an OpenShift/ROKS cluster. Plain Kubernetes is simpler;
the ROKS-specific workarounds are called out explicitly.

### 1. Download karmadactl

```bash
KARMADA_VERSION=v1.19.0
curl -L "https://github.com/karmada-io/karmada/releases/download/${KARMADA_VERSION}/karmadactl-linux-arm64.tgz" \
  -o /tmp/karmadactl.tgz
tar -xzf /tmp/karmadactl.tgz -C /tmp/
chmod +x /tmp/karmadactl
/tmp/karmadactl version
```

Use `linux-amd64` on x86 machines; `linux-arm64` on Apple Silicon / aarch64.

### 2. Pre-create the karmada-system namespace and grant OpenShift SCCs

On OpenShift, containers must be granted `anyuid` and `privileged` SCCs or etcd and
the API server containers won't start (they run as root).

```bash
kubectl create namespace karmada-system

kubectl create clusterrolebinding karmada-system-anyuid \
  --clusterrole=system:openshift:scc:anyuid \
  --group=system:serviceaccounts:karmada-system

kubectl create clusterrolebinding karmada-system-privileged \
  --clusterrole=system:openshift:scc:privileged \
  --group=system:serviceaccounts:karmada-system
```

**Not needed on plain Kubernetes.**

### 3. Pre-create a LoadBalancer service to get an external hostname

This gives you the external hostname to embed in the TLS certificate before running init.
IBM Cloud ROKS assigns a hostname (not a bare IP) to LoadBalancer services.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: karmada-apiserver-probe-lb
  namespace: karmada-system
spec:
  type: LoadBalancer
  ports:
  - port: 5443
    targetPort: 5443
  selector:
    app: karmada-apiserver
EOF

# Wait for hostname
kubectl get svc karmada-apiserver-probe-lb -n karmada-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Record the hostname (e.g. `f21f365b-eu-de.lb.appdomain.cloud`).

### 4. Run karmadactl init

**On ROKS, the init will fail partway through** because it tries to connect to the
karmada-apiserver via a NodePort on a private VPC IP (not reachable from outside the VPC).
This is expected. Run it anyway — it successfully deploys etcd, karmada-apiserver, and
karmada-aggregated-apiserver before failing.

```bash
mkdir -p ~/karmada

/tmp/karmadactl init \
  --namespace karmada-system \
  --karmada-data ~/karmada \
  --karmada-pki ~/karmada/pki \
  --etcd-storage-mode PVC \
  --storage-classes-name ibmc-vpc-block-10iops-tier \
  --karmada-apiserver-replicas 1 \
  --etcd-replicas 1 \
  --cert-external-dns <YOUR-LB-HOSTNAME> \
  --wait-component-ready-timeout 300
```

Expected failure:
```
F deploy.go:74] unable to create Namespace: Post "https://10.x.x.x:32443/...": i/o timeout
```

Verify what was deployed:
```bash
kubectl get pods -n karmada-system
# Should show: etcd-0, karmada-apiserver-*, karmada-aggregated-apiserver-*
```

**On plain Kubernetes** where NodePorts are reachable, init completes fully and you can
skip steps 5–10.

### 5. Port-forward the karmada-apiserver to complete setup

```bash
kubectl port-forward svc/karmada-apiserver 5443:5443 -n karmada-system --address 127.0.0.1 &

# Create a local kubeconfig pointing at localhost
cp ~/karmada/karmada-apiserver.config ~/karmada/karmada-local.config
sed -i 's|https://10\.[0-9.]*:32443|https://127.0.0.1:5443|g' ~/karmada/karmada-local.config

# Verify connectivity
kubectl --kubeconfig ~/karmada/karmada-local.config get ns
```

### 6. Install Karmada CRDs

```bash
kubectl --kubeconfig ~/karmada/karmada-local.config \
  apply -f ~/karmada/crds/bases/ --recursive
```

### 7. Create required namespaces in the karmada-apiserver

The karmada-apiserver has its own namespace set (separate from the host cluster).

```bash
for ns in karmada-system karmada-cluster karmada; do
  kubectl --kubeconfig ~/karmada/karmada-local.config create namespace $ns
done
```

### 8. Create secrets and deploy remaining control-plane components

The karmadactl init skipped deploying the controller-manager, scheduler, and webhook.
Deploy them manually.

```bash
# In-cluster kubeconfig (components inside the cluster use the service DNS name)
cp ~/karmada/karmada-local.config ~/karmada/karmada-incluster.config
sed -i 's|https://127.0.0.1:5443|https://karmada-apiserver.karmada-system.svc:5443|g' \
  ~/karmada/karmada-incluster.config

# Create secrets
kubectl create secret generic karmada-kubeconfig -n karmada-system \
  --from-file=karmada-apiserver.config=~/karmada/karmada-incluster.config

kubectl create secret generic karmada-cert -n karmada-system \
  --from-file=ca.crt=~/karmada/pki/ca.crt \
  --from-file=tls.crt=~/karmada/pki/karmada.crt \
  --from-file=tls.key=~/karmada/pki/karmada.key
```

Deploy the three components (note: `--bind-address` and `--tls-cert-file` are NOT valid
flags in v1.19.0 — use the flags shown below):

```bash
cat <<'EOF' | kubectl apply -f -
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-controller-manager
  namespace: karmada-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: karmada-controller-manager
  template:
    metadata:
      labels:
        app: karmada-controller-manager
    spec:
      containers:
      - name: karmada-controller-manager
        image: docker.io/karmada/karmada-controller-manager:v1.19.0
        command:
        - /bin/karmada-controller-manager
        - --kubeconfig=/etc/karmada/config/karmada-apiserver.config
        - --cluster-status-update-frequency=10s
        - --v=4
        volumeMounts:
        - name: karmada-config
          mountPath: /etc/karmada/config
      volumes:
      - name: karmada-config
        secret:
          secretName: karmada-kubeconfig
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-scheduler
  namespace: karmada-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: karmada-scheduler
  template:
    metadata:
      labels:
        app: karmada-scheduler
    spec:
      containers:
      - name: karmada-scheduler
        image: docker.io/karmada/karmada-scheduler:v1.19.0
        command:
        - /bin/karmada-scheduler
        - --kubeconfig=/etc/karmada/config/karmada-apiserver.config
        - --v=4
        volumeMounts:
        - name: karmada-config
          mountPath: /etc/karmada/config
      volumes:
      - name: karmada-config
        secret:
          secretName: karmada-kubeconfig
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-webhook
  namespace: karmada-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: karmada-webhook
  template:
    metadata:
      labels:
        app: karmada-webhook
    spec:
      containers:
      - name: karmada-webhook
        image: docker.io/karmada/karmada-webhook:v1.19.0
        command:
        - /bin/karmada-webhook
        - --kubeconfig=/etc/karmada/config/karmada-apiserver.config
        - --cert-dir=/etc/karmada/pki
        - --tls-cert-file-name=tls.crt
        - --tls-private-key-file-name=tls.key
        - --v=4
        volumeMounts:
        - name: karmada-config
          mountPath: /etc/karmada/config
        - name: karmada-cert
          mountPath: /etc/karmada/pki
      volumes:
      - name: karmada-config
        secret:
          secretName: karmada-kubeconfig
      - name: karmada-cert
        secret:
          secretName: karmada-cert
---
apiVersion: v1
kind: Service
metadata:
  name: karmada-webhook
  namespace: karmada-system
spec:
  selector:
    app: karmada-webhook
  ports:
  - port: 443
    targetPort: 8443
EOF
```

### 9. Register the aggregated-apiserver (ROKS-specific workaround)

The karmada-aggregated-apiserver serves the `cluster.karmada.io/v1alpha1` API (the
`Cluster` resource used to register member clusters). It needs to be registered as an
APIService with the karmada-apiserver.

The kube-aggregator routing code uses the Service's ClusterIP. On ROKS the karmada-apiserver
has its own virtual service CIDR (`10.96.0.0/12`) with no kube-proxy, so virtual ClusterIPs
are unreachable. The fix is to use an `ExternalName` service — the kube-aggregator has an
explicit code path for ExternalName that uses the DNS name directly, bypassing ClusterIP.

```bash
# Create ExternalName service in karmada namespace pointing to the host cluster service
cat <<'EOF' | kubectl --kubeconfig ~/karmada/karmada-local.config apply -f -
apiVersion: v1
kind: Service
metadata:
  name: karmada-aggregated-apiserver
  namespace: karmada-system
spec:
  type: ExternalName
  externalName: karmada-aggregated-apiserver.karmada-system.svc.cluster.local
  ports:
  - port: 443
    protocol: TCP
EOF

# Register the APIService
cat <<'EOF' | kubectl --kubeconfig ~/karmada/karmada-local.config apply -f -
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1alpha1.cluster.karmada.io
spec:
  insecureSkipTLSVerify: true
  group: cluster.karmada.io
  groupPriorityMinimum: 2000
  service:
    name: karmada-aggregated-apiserver
    namespace: karmada-system
    port: 443
  version: v1alpha1
  versionPriority: 10
EOF

# Verify: should show "Available - True - all checks passed"
kubectl --kubeconfig ~/karmada/karmada-local.config \
  get apiservice v1alpha1.cluster.karmada.io \
  -o jsonpath='{.status.conditions[0].message}'
```

**On plain Kubernetes** (where NodePorts are reachable and `karmadactl init` completes),
this step is handled automatically by init.

### 10. Join a member cluster

```bash
MEMBER_CONTEXT='<your-member-cluster-context>'

/tmp/karmadactl join <cluster-name> \
  --kubeconfig ~/karmada/karmada-local.config \
  --cluster-kubeconfig ~/.kube/config \
  --cluster-context "${MEMBER_CONTEXT}" \
  --cluster-namespace karmada-system
```

On ROKS, the controller-manager health-checks the member cluster via
`spec.apiEndpoint`, which defaults to the external API server URL. From inside the
pod network this is unreachable. Patch it to the internal Kubernetes service IP:

```bash
KUBE_SVC_IP=$(kubectl get svc kubernetes -n default -o jsonpath='{.spec.clusterIP}')

# Patch the Cluster object in the karmada-apiserver
kubectl --kubeconfig ~/karmada/karmada-local.config patch cluster <cluster-name> \
  --type='merge' \
  -p "{\"spec\":{\"apiEndpoint\":\"https://${KUBE_SVC_IP}\"}}"

# Patch the member kubeconfig secret
CURRENT_KC=$(kubectl --kubeconfig ~/karmada/karmada-local.config \
  get secret <cluster-name> -n karmada-system \
  -o jsonpath='{.data.kubeconfig}' | base64 -d)
NEW_KC=$(echo "$CURRENT_KC" | sed \
  "s|server: https://.*|server: https://${KUBE_SVC_IP}|")
kubectl --kubeconfig ~/karmada/karmada-local.config \
  patch secret <cluster-name> -n karmada-system \
  --type='json' \
  -p="[{\"op\":\"replace\",\"path\":\"/data/kubeconfig\",\"value\":\"$(echo $NEW_KC | base64 -w 0)\"}]"
```

Verify:
```bash
kubectl --kubeconfig ~/karmada/karmada-local.config get cluster
# NAME       VERSION    MODE   READY
# infraops   v1.33.13   Push   True
```

---

## The Actual Test

### Create a test CRD and CR on the Karmada control plane

```bash
# CRD on the karmada-apiserver
cat <<'EOF' | kubectl --kubeconfig ~/karmada/karmada-local.config apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: widgetjobs.spike.example.com
spec:
  group: spike.example.com
  names:
    kind: WidgetJob
    plural: widgetjobs
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              image:
                type: string
              replicas:
                type: integer
              gpuResources:
                type: string
EOF

# A CR instance
cat <<'EOF' | kubectl --kubeconfig ~/karmada/karmada-local.config apply -f -
apiVersion: spike.example.com/v1
kind: WidgetJob
metadata:
  name: test-widgetjob
  namespace: default
spec:
  image: nvidia/cuda:12.0-base
  replicas: 2
  gpuResources: "nvidia.com/gpu=1"
EOF

# PropagationPolicy targeting the member cluster
cat <<'EOF' | kubectl --kubeconfig ~/karmada/karmada-local.config apply -f -
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: widgetjob-propagation
  namespace: default
spec:
  resourceSelectors:
  - apiVersion: spike.example.com/v1
    kind: WidgetJob
  placement:
    clusterAffinity:
      clusterNames:
      - infraops
EOF
```

**Important:** The CRD must also be installed on the member cluster. If it is not,
the Karmada scheduler refuses to place the workload (see Finding 2 below).

```bash
# Install the same CRD on the member cluster
kubectl apply -f widgetjob-crd.yaml
```

---

## Findings

### Finding 1: The full CR is propagated as-is — no decomposition

Inspect the `Work` object created in the member cluster's execution namespace:

```bash
kubectl --kubeconfig ~/karmada/karmada-local.config \
  get works -n karmada-es-infraops

# NAME                         WORKLOAD-KIND   APPLIED
# test-widgetjob-57d5b44989   WidgetJob       True
```

The Work manifest contains the complete raw CR:

```yaml
# kubectl --kubeconfig ~/karmada/karmada-local.config \
#   get work test-widgetjob-57d5b44989 -n karmada-es-infraops -o yaml
apiVersion: work.karmada.io/v1alpha1
kind: Work
metadata:
  name: test-widgetjob-57d5b44989
  namespace: karmada-es-infraops
spec:
  workload:
    manifests:
    - apiVersion: spike.example.com/v1
      kind: WidgetJob           # <-- still a WidgetJob, not Pods
      metadata:
        name: test-widgetjob
        namespace: default
      spec:
        gpuResources: nvidia.com/gpu=1
        image: nvidia/cuda:12.0-base
        replicas: 2             # <-- spec fields unchanged
```

And on the member cluster after the Work is applied:

```bash
kubectl get widgetjob test-widgetjob -n default -o yaml
# apiVersion: spike.example.com/v1
# kind: WidgetJob
# spec:
#   gpuResources: nvidia.com/gpu=1
#   image: nvidia/cuda:12.0-base
#   replicas: 2
```

The CR is created verbatim on the member cluster. No pods, no Deployments, no
native Kubernetes objects. The Karmada execution controller is a dumb `kubectl apply`
of the manifest embedded in the Work object.

### Finding 2: The scheduler checks for the CRD on the member cluster

Before placing a workload, the Karmada scheduler verifies the required API is
registered on the target cluster. If the CRD is absent, scheduling is refused:

```
Cluster(infraops) not fit as missing API(spike.example.com/v1, kind=WidgetJob)
ScheduleBindingFailed: 0/1 clusters are available: 1 cluster(s) did not have the API resource.
```

This means the member cluster needs at minimum the **CRD** installed. Without it,
Karmada won't propagate to that cluster at all.

### Finding 3: The Resource Interpreter is for scheduling metadata only

The controller-manager logs confirm the interpreter's role:

```
Default interpreter is not enabled for kind "spike.example.com/v1, Kind=WidgetJob"
with operation "InterpretReplica".
Hook interpreter is not enabled for kind "spike.example.com/v1, Kind=WidgetJob"
with operation "InterpretReplica".
```

The interpreter tries to extract `replicas` from the CR for scheduling decisions
(cluster capacity planning). When no interpreter is registered, scheduling still
proceeds — the CR is propagated without replica-aware placement. The interpreter
does **not** decompose the CR; it only provides hints to the scheduler.

The RayJob interpreter PR (#6947) adds logic for `InterpretReplicas`,
`ReviseReplica`, `InterpretHealth` — all metadata extraction, not decomposition.

---

## Conclusion

**Berto's hypothesis is refuted.**

Karmada does not decompose CRs into native Kubernetes objects. The Resource
Interpreter is purely a scheduling metadata layer. The full CR — `TrainJob`,
`RayJob`, or any other custom resource — is embedded in the `Work` object and
applied verbatim to the member cluster.

**Implication for our architecture:**
GPU worker clusters must have the relevant operators installed
(training-operator for TrainJob, KubeRay for RayJob, etc.).
A worker cluster without the operator will have the CR applied but nothing will
happen — the CR will just sit there unreconciled.

This does not make Karmada valueless. It still provides:

- **Resource-aware cluster selection** — the scheduler picks the cluster with
  capacity for the workload (if an interpreter provides replica/resource hints)
- **Automated CRD propagation** — Karmada can propagate the CRDs themselves to
  member clusters via `ClusterPropagationPolicy`, reducing operator bootstrapping
  effort
- **Placement policies** — cluster affinity, taints/tolerations, failover
- **Dependency tracking** — ConfigMaps/Secrets travel with workloads
- **Override policies** — per-cluster spec overrides without changing the source CR

The question of whether Karmada is worth the complexity compared to multi-kueue
remains open and depends on how much value the above features provide versus the
operational overhead of running a Karmada control plane.

---

## Broader Architecture Context

This spike is part of a larger question: how to provide isolated RHOAI environments
with shared GPU capacity.

| Approach | Isolation | GPU scheduling | RHOAI compat | Complexity |
|----------|-----------|----------------|--------------|------------|
| vCluster + KAI | Per-tenant virtual cluster | On-demand (KAI/Kueue) | Needs vcluster-openshift sidecar | High |
| KubeVirt + SNC | Full VM isolation | Lease-based (no preemption) | Native OCP | Low |
| Karmada | Per-cluster | On-demand (Kueue on workers) | Needs operators on workers | Medium |
| Single cluster + Kueue | Namespace only | On-demand | One RHOAI version | Low |
| FIFO cluster pool | Full cluster | Lease-based | Native OCP | Low |

A hybrid approach — Kueue for shared ML workloads on a single cluster, plus a small
FIFO pool of full clusters for isolated RHOAI testing — may be the pragmatic middle
ground. No single approach gives isolation + on-demand GPU + RHOAI compatibility
without significant complexity.

---

## Cleanup

```bash
# Remove Karmada from the host cluster
kubectl delete namespace karmada-system
kubectl delete clusterrolebinding karmada-system-anyuid karmada-system-privileged
kubectl delete crd widgetjobs.spike.example.com
kubectl delete clusterrolebinding karmada-impersonator 2>/dev/null || true

# Remove the member cluster registration artifacts
kubectl delete namespace karmada karmada-cluster 2>/dev/null || true

# Remove the karmada data directory
rm -rf ~/karmada

# If the member cluster was real infraops:
# kubectl --context infraops delete namespace karmada-system
# kubectl --context infraops delete crd widgetjobs.spike.example.com
```
