# Installing Hyperstack CSI Driver for Self-Managed Multi-Cloud Kubernetes

**For:** Self-managed K3S clusters with AWS control plane + Hyperstack GPU workers
**Time:** ~30 minutes
**Prerequisites:** Cluster running, kubectl configured, Helm installed

---

## Step 1: Get Your Hyperstack Information

### Get API Key

1. Login to Hyperstack console: https://console.hyperstack.cloud/
2. Navigate to API Keys
3. Create new API key (save it securely)

Example: `e6789ccb-2410-4be4-94df-7934fbc4a602`

### Get Environment Name

```bash
curl -X GET "https://infrahub-api.nexgencloud.com/v1/core/environments" \
  -H "api_key: YOUR_API_KEY" | jq -r '.environments[] | select(.region=="CANADA-1") | .name'
```

Example output: `inferx-ev`

### Get Instance IDs

```bash
curl -X GET "https://infrahub-api.nexgencloud.com/v1/core/virtual-machines" \
  -H "api_key: YOUR_API_KEY" | jq '.instances[] | {id, name}'
```

Example output:
```json
{
  "id": 447048,
  "name": "gpu-workerx"
}
```

**Save these values:**
- API Key: `e6789ccb-2410-4be4-94df-7934fbc4a602`
- Environment: `inferx-ev`
- Instance ID: `447048`
- Instance Name: `gpu-workerx`

---

## Step 2: Label Hyperstack Nodes

**Critical:** Node name in Kubernetes MUST match Hyperstack VM name exactly.

Check your node names:
```bash
kubectl get nodes
```

If your Hyperstack node is named `gpu-workerx-a7k-h100-2` but Hyperstack VM is `gpu-workerx`, the CSI won't work. Fix your cloud-init to use exact VM names.

Once names match, add required labels:

```bash
kubectl label node gpu-workerx cloud=hyperstack
kubectl label node gpu-workerx hyperstack.cloud/instance-id=447048
kubectl label node gpu-workerx minio-node=true
```

Verify:
```bash
kubectl get node gpu-workerx -o jsonpath='{.metadata.labels}' | grep hyperstack
```

---

## Step 3: Clone Remodl AI Fork

```bash
git clone https://github.com/remodlai/csi-hyperstack.git
cd csi-hyperstack
```

---

## Step 4: Build Custom CSI Image

```bash
docker buildx build \
  --platform linux/amd64 \
  -t remodlai/csi-hyperstack:v0.0.6-remodl2 \
  -f Dockerfile \
  --build-arg VERSION=v0.0.6-remodl2 \
  --output type=docker \
  .
```

Push to Docker Hub (or your private registry):

```bash
docker push remodlai/csi-hyperstack:v0.0.6-remodl2
```

---

## Step 5: Create Values File

Create `csi-values.yaml`:

```yaml
hyperstack:
  apiKey: "e6789ccb-2410-4be4-94df-7934fbc4a602"  # Your API key

nodeSelector:
  cloud: hyperstack

tolerations:
- key: nvidia.com/gpu
  operator: Exists
  effect: NoSchedule

components:
  csiHyperstack:
    image: remodlai/csi-hyperstack
    tag: v0.0.6-remodl2
```

---

## Step 6: Install CSI Driver

```bash
helm install csi-hyperstack ./charts/csi-hyperstack \
  --namespace kube-system \
  -f csi-values.yaml
```

### Verify Installation

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=csi-hyperstack
```

**Expected output:**
```
NAME                              READY   STATUS    AGE   NODE
csi-hyperstack-55b5f7c658-xxxxx   4/4     Running   30s   aws-cpu-worker-1
csi-hyperstack-node-xwzbj         3/3     Running   30s   gpu-workerx
```

**Key check:** Node pod ONLY on `gpu-workerx`, not on AWS nodes.

If node pod is crashing, check logs:
```bash
kubectl logs -n kube-system csi-hyperstack-node-xxxxx -c csi-hyperstack-node
```

Common errors:
- `nodes "gpu-workerx-random" not found` → Node name doesn't match VM name
- `label hyperstack.cloud/instance-id not found` → Missing instance ID label
- Crashes on AWS nodes → nodeSelector didn't apply

---

## Step 7: Create Custom Storage Class

**Critical:** Upstream storage class won't work for self-managed clusters.

Create `hyperstack-storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-hyperstack-inferx
provisioner: hyperstack.csi.nexgencloud.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: Cloud-SSD
  environment: inferx-ev  # YOUR environment name from Step 1
reclaimPolicy: Delete
allowVolumeExpansion: true
```

Apply:
```bash
kubectl apply -f hyperstack-storageclass.yaml
```

Verify:
```bash
kubectl get storageclass csi-hyperstack-inferx
```

---

## Step 8: Test Volume Provisioning

Create test PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: csi-hyperstack-inferx
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: default
spec:
  nodeSelector:
    cloud: hyperstack
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: test-pvc
```

Apply and watch:
```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc test-pvc --watch
```

**Expected progression:**
1. PVC: Pending (waiting for pod)
2. Pod schedules on gpu-workerx
3. CSI provisions Hyperstack volume
4. PVC: Bound
5. Pod: Running

**If PVC stays Pending:**
```bash
kubectl describe pvc test-pvc
```

Look for provisioning errors in Events section.

---

## Step 9: Fix Permission Issue (Current Workaround)

**Known Issue:** CSI driver doesn't honor `fsGroup` security context. Volumes mount as `root:root` even when pod specifies `fsGroup: 1000`.

**Symptoms:**
- Pod crashes with "file access denied"
- Application can't write to mounted volume

**Temporary Fix:**

```bash
# Get pod UID from describe
kubectl describe pod <pod-name> -n <namespace> | grep UID

# SSH to node
ssh ubuntu@<node-tailscale-ip>

# Find mount path (replace with actual PVC ID)
sudo find /var/lib/kubelet/pods -name "pvc-*" -type d 2>/dev/null

# Fix ownership (replace with actual path and UID:GID your app needs)
sudo chown -R 1000:1000 /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pvc-id>/mount

# Delete pod to restart
kubectl delete pod <pod-name> -n <namespace>
```

**Proper Fix (TODO):**
Add fsGroup support to CSI node server. The driver needs to:
1. Read fsGroup from VolumeContext
2. Apply ownership during NodeStageVolume
3. Test: https://github.com/kubernetes-csi/csi-test/tree/master/pkg/sanity

This is tracked as tech debt in dev-notes.

---

## Step 10: Package Helm Chart to Harbor

```bash
# Package chart
helm package charts/csi-hyperstack -d /tmp

# Push to Harbor OCI registry
helm push /tmp/csi-hyperstack-0.0.8.tgz oci://harbor.tail972298.ts.net/inferx
```

---

## Complete Example: MinIO on Hyperstack CSI

This is the actual production configuration for MinIO-Hyperstack tenant:

```yaml
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio-hyperstack
  namespace: minio-hyperstack
spec:
  image: quay.io/minio/minio:latest
  pools:
  - name: pool-0
    servers: 1
    volumesPerServer: 1
    volumeClaimTemplate:
      metadata:
        name: data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1500Gi
        storageClassName: csi-hyperstack-inferx  # Use custom storage class
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: minio-node
              operator: In
              values:
              - "true"
    tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      fsGroup: 1000  # Should work but doesn't due to CSI bug
      runAsNonRoot: true
```

After deployment, if pod crashes with permission errors:

1. Find the mount path on node
2. `sudo chown -R 1000:1000 <mount-path>`
3. Delete pod to restart

---

## Troubleshooting

### CSI Node Pod Crashes on AWS Nodes

**Symptom:** `csi-hyperstack-node` pods crash on non-Hyperstack nodes

**Fix:** Verify `nodeSelector.cloud=hyperstack` in values and DaemonSet template includes:
```yaml
nodeSelector:
  cloud: hyperstack
```

### "label hyperstack.cloud/instance-id not found"

**Symptom:** CSI node pod logs show missing instance ID label

**Fix:**
```bash
kubectl label node <node-name> hyperstack.cloud/instance-id=<vm-id>
```

Get VM ID from Hyperstack API (Step 1).

### "strconv.Atoi: parsing empty string"

**Symptom:** PVC provisioning fails with parse error

**Fix:** Storage class is missing `environment` parameter or using wrong storage class.

Verify storage class has:
```yaml
parameters:
  environment: your-env-name
```

### "fsGroup not working" / Permission Denied

**Symptom:** Pod can't write to volume even with `fsGroup: 1000`

**Current Status:** Known bug - CSI doesn't apply fsGroup during mount

**Workaround:** Manual chown after first mount (documented above)

**Permanent Fix:** Add fsGroup support to NodeStageVolume implementation

---

## What Works

✅ Dynamic volume provisioning on Hyperstack
✅ Volume attachment to correct VM
✅ Node targeting (runs only on Hyperstack nodes)
✅ Environment parameter for self-managed clusters
✅ Multiple volumes per node (up to 10)
✅ Volume expansion

## Known Limitations

❌ fsGroup not honored (requires manual chown)
❌ ReadWriteOnce only (Hyperstack volume limit)
❌ 10 volume per node limit (Hyperstack API limit)
❌ Requires exact node name matching

---

## Images

**Remodl AI Custom Images:**
- `remodlai/csi-hyperstack:v0.0.6-remodl2` (with environment parameter support)

**Upstream Images:**
- `reg.digitalocean.ngbackend.cloud/hyperstack-csi-driver/csi:v0.0.1` (won't work for self-managed)

---

## Quick Start Summary

```bash
# 1. Label nodes
kubectl label node gpu-workerx cloud=hyperstack
kubectl label node gpu-workerx hyperstack.cloud/instance-id=447048

# 2. Install CSI
helm install csi-hyperstack ./charts/csi-hyperstack \
  --namespace kube-system \
  --set hyperstack.apiKey=YOUR_KEY \
  --set nodeSelector.cloud=hyperstack \
  --set tolerations[0].key=nvidia.com/gpu \
  --set components.csiHyperstack.image=remodlai/csi-hyperstack \
  --set components.csiHyperstack.tag=v0.0.6-remodl2

# 3. Create storage class
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: csi-hyperstack-inferx
provisioner: hyperstack.csi.nexgencloud.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: Cloud-SSD
  environment: inferx-ev
reclaimPolicy: Delete
EOF

# 4. Use in PVCs
storageClassName: csi-hyperstack-inferx

# 5. If permission errors, fix ownership manually (one-time per PVC)
```
