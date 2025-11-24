# Technical Debt and Known Issues

## Issue #1: fsGroup Not Honored During Volume Mount

**Severity:** Medium
**Impact:** Requires manual intervention after volume provisioning
**Status:** Workaround exists, permanent fix needed

### The Problem

When a Pod specifies `securityContext.fsGroup`, Kubernetes should automatically set ownership of mounted volumes to that group ID. The Hyperstack CSI driver does not honor this field, resulting in volumes mounted as `root:root` regardless of the Pod's security context.

**Example:**
```yaml
securityContext:
  fsGroup: 1000  # Should make volume owned by GID 1000
  runAsUser: 1000
  runAsGroup: 1000
```

**Actual Result:**
```bash
$ ls -ld /export
drwxr-xr-x 3 root root 4096 Nov 24 20:05 /export
```

**Expected Result:**
```bash
$ ls -ld /export
drwxrwsr-x 3 1000 1000 4096 Nov 24 20:05 /export
```

### Current Workaround

After the first Pod attempts to start and fails with permission errors:

1. SSH to the node where the Pod is scheduled
2. Find the volume mount path:
   ```bash
   kubectl describe pod <pod-name> -n <namespace> | grep "pvc-"
   # Note the PVC ID (e.g., pvc-e59fddff-acbb-4f0e-8eb6-ae203763c4b9)
   ```

3. Locate the mount on the node:
   ```bash
   mount | grep <pvc-id>
   # Shows: /dev/vdX on /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pvc-id>/mount
   ```

4. Fix ownership:
   ```bash
   sudo chown -R 1000:1000 /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~csi/<pvc-id>/mount
   ```

5. Delete the Pod to restart:
   ```bash
   kubectl delete pod <pod-name> -n <namespace>
   ```

The StatefulSet or Deployment controller recreates the Pod, which now starts successfully.

### Permanent Fix Required

The CSI driver's `NodeStageVolume` and `NodePublishVolume` implementations need to read `VolumeContext` for `csi.storage.k8s.io/fsGroup` and apply ownership.

**Implementation Location:** `pkg/driver/nodeserver.go`

**Required Changes:**

1. Extract fsGroup from VolumeContext:
```go
func (ns *NodeServer) NodeStageVolume(ctx context.Context, req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {
    // ... existing code ...

    // Read fsGroup from volume context
    fsGroupStr := req.VolumeContext["csi.storage.k8s.io/fsGroup"]
    if fsGroupStr != "" {
        fsGroup, err := strconv.Atoi(fsGroupStr)
        if err != nil {
            return nil, status.Errorf(codes.Internal, "invalid fsGroup: %v", err)
        }

        // Apply ownership after mount
        if err := os.Chown(stagingPath, 0, fsGroup); err != nil {
            return nil, status.Errorf(codes.Internal, "failed to chown: %v", err)
        }

        // Set setgid bit for new files to inherit group
        if err := os.Chmod(stagingPath, 0775|os.ModeSetgid); err != nil {
            return nil, status.Errorf(codes.Internal, "failed to chmod: %v", err)
        }
    }

    // ... rest of implementation ...
}
```

2. Add tests using CSI sanity test suite:
```go
// Test fsGroup gets applied
// Test non-fsGroup pods still work
// Test fsGroup changes between mounts
```

3. Update documentation to remove workaround steps

### Why This Matters

Applications running as non-root users (MinIO, databases, etc.) cannot start without this fix. The manual workaround:
- Requires SSH access to nodes (security concern)
- Doesn't scale (manual for each PVC)
- Breaks automation (can't deploy via GitOps)
- Error-prone (easy to forget or misconfigure)

Production deployments need fsGroup support for stateful applications to work reliably.

### Testing the Fix

After implementing fsGroup support:

1. Deploy a Pod with `fsGroup: 1234`
2. Verify volume is owned by GID 1234 without manual intervention
3. Verify Pod starts and can write to volume
4. Test with fsGroup changes (redeploy Pod with different fsGroup)
5. Run CSI sanity tests: https://github.com/kubernetes-csi/csi-test

---

## Issue #2: No Multi-Cloud Configuration in Upstream

**Severity:** High (for multi-cloud deployments)
**Status:** Fixed in Remodl fork

Documented for completeness - already resolved in this fork.

Upstream driver lacks:
- nodeSelector support
- Tolerations support
- Environment parameter override

All fixed in Remodl AI fork via Helm template modifications and controller code changes.

---

## Contributing Upstream

Once fsGroup support is implemented and tested, consider contributing it back to NexGenCloud upstream:

1. Create clean branch with fsGroup changes only
2. Write tests demonstrating the fix
3. Submit PR to https://github.com/NexGenCloud/csi-hyperstack
4. Reference Kubernetes CSI spec for fsGroup: https://kubernetes-csi.github.io/docs/support-fsgroup.html

If upstream accepts, future versions won't need the manual workaround.
