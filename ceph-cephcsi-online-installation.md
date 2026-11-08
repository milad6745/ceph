# Ceph Squid + Ceph CSI RBD on Kubernetes — Online Installation Runbook

> **Scope:** Git-ready, repeatable **internet-connected** installation guide for an external Ceph cluster and the Ceph CSI RBD driver on Kubernetes.  
> **Reference environment:** Ubuntu 24.04; Ceph **Squid 19.2.x** (tested example: 19.2.3); Kubernetes **v1.37.1**; Helm 3; Ceph CSI **v3.18.1** (validate compatibility on Kubernetes 1.37 before production).  
> **Status:** Installation recipe, **not a claim that all CSI/PVC steps were completed in the source lab**. In the source lab Ceph, a single OSD, and RBD pool were verified; CSI deployment / PVC mount validation was the next step.

## 0. Architecture, assumptions and safety

```text
Kubernetes cluster (control plane + workers)
  └── Ceph CSI RBD controller (Deployment) + node plugin (DaemonSet)
       └── Ceph MONs (TCP 3300; legacy 6789 if enabled)
            └── Ceph OSDs (configured OSD service ports)
                 └── RBD pool: kubernetes-rbd
```

- Ceph is **external to Kubernetes**. Ceph daemons run in containers managed by **cephadm** and Podman/Docker, **not** Kubernetes/containerd.
- Use separate, unused **block devices** for OSDs. **Never** run `ceph orch apply osd --all-available-devices` on a mixed-use host without reviewing the discovered devices.
- For resilient deployment, plan at least **three failure domains** (typically three hosts, MON quorum, and OSD replicas); use a Ceph pool with `size=3`, `min_size=2` as an example, subject to capacity and architecture review. Single-host `size=1` is **lab-only** and provides no replica.
- Expose Ceph MON and OSD ports only on permitted networks, ensure all CSI controllers and mounting nodes can connect, and synchronize time using trusted NTP/Chrony.
- Use an internal CA-signed HTTPS certificate for the Dashboard. The default self-signed certificate causes browser warnings.
- **Do not commit Ceph keyrings, Kubernetes Secrets, passwords, or secret-bearing command outputs to Git.**

### Example environment variables

Change these for your own installation; the IPs below are examples from the lab.

```bash
export CEPH_MON_IP=192.168.72.50
export CEPH_HOST=ceph
export CEPH_POOL=kubernetes-rbd
export CSI_NAMESPACE=ceph-csi
export CSI_USER=kubernetes-rbd
```

## 1. Install cephadm on the internet-connected Ceph host

On a **clean Ubuntu 24.04 Ceph server**, as root:

```bash
apt-get update
apt-get install -y curl ca-certificates chrony podman lvm2 openssh-server
systemctl enable --now chrony ssh
chronyc tracking
podman --version
```

Use **one** cephadm installation method, not a mixture. The official Squid instructions support obtaining a standalone cephadm for a pinned release, then installing it as a package:

```bash
CEPH_VERSION=19.2.3
curl -fL "https://download.ceph.com/rpm-${CEPH_VERSION}/el9/noarch/cephadm" -o cephadm
chmod +x cephadm
./cephadm add-repo --release squid
./cephadm install
which cephadm
cephadm version
```

Alternatively, on a distribution with a properly maintained Squid package source, use `apt-get install cephadm`. **Verify the resulting version** and do not assume Ubuntu's default repository ships 19.2.3.

Install CLI tooling:

```bash
cephadm install ceph-common
ceph --version
```

## 2. Bootstrap Ceph

For the **online, multi-host path**, don't use single-host defaults:

```bash
cephadm bootstrap --mon-ip "${CEPH_MON_IP}"
ceph -s
ceph orch host ls
ceph orch ps
```

The bootstrap provides `/etc/ceph/ceph.conf` and the initial admin keyring on the bootstrap host. Store the initial Dashboard admin password in a password manager; rotate bootstrap credentials.

For a **single-server lab only**, Ceph can instead be bootstrapped with:

```bash
# LAB ONLY; use instead of the bootstrap above, not in addition to it
# cephadm bootstrap --mon-ip "$CEPH_MON_IP" \
#   --single-host-defaults --skip-monitoring-stack
```

On the verified lab, `--skip-monitoring-stack` omitted the default monitoring stack; for production deploy metrics/alerting and test backups and recovery.

### Adding Ceph hosts for a resilient deployment

Distribute cephadm's SSH public key, then add hosts:

```bash
ceph cephadm get-pub-key > /root/ceph.pub
# Install /root/ceph.pub into the target host's authorized_keys securely.
ceph orch host add ceph02 <CEPH02_IP>
ceph orch host add ceph03 <CEPH03_IP>
ceph orch host ls
```

Use distinct hosts/failure domains, plan MON placement and CRUSH topology, and verify time synchronization before deploying OSDs.

## 3. Provision OSDs with real block devices

First inspect devices **on the Ceph hosts**:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
ceph orch device ls --wide --refresh
```

Verify the dedicated device (for example `/dev/sdb`) is **not mounted, partitioned for another purpose, or holding data**. OSD provisioning is destructive to its target. Add the selected device explicitly:

```bash
# EXAMPLE ONLY: replace device and host after verifying they are safe
ceph orch daemon add osd ceph:/dev/sdb
ceph osd tree
ceph orch ps --daemon_type osd
ceph -s
```

For a production deployment, repeat on additional Ceph hosts using dedicated disks; choose a suitable CRUSH rule and replication level.

### Lab-only alternative: sparse-file / loop / LVM OSD

The original single-VM lab did not have spare storage, so it used an **80 GiB sparse backing file** and a **70 GiB LVM logical volume**. This is unsuitable for durable production storage and may consume the OS root filesystem unexpectedly.

```bash
# LAB ONLY — run only on a disposable host with enough root filesystem space
mkdir -p /var/lib/ceph-lab/disks
truncate -s 80G /var/lib/ceph-lab/disks/osd1.img
LOOP_DEV=$(losetup --find --show /var/lib/ceph-lab/disks/osd1.img)
pvcreate "$LOOP_DEV"
vgcreate ceph_lab_vg "$LOOP_DEV"
lvcreate -L 70G -n osd1 ceph_lab_vg
ceph orch daemon add osd ceph:/dev/ceph_lab_vg/osd1
ceph -s
```

**Never repeat `pvcreate`, `vgcreate`, or `ceph orch daemon add osd` against the existing populated LV.** The backing file / loop must be reattached and the VG activated *before* the OSD unit is started after reboot. The verified lab used a dedicated idempotent shell script and a systemd drop-in `ExecStartPre` for its specific `ceph-<FSID>@osd.0.service`. This workaround should **not** be copied into a production deployment with real disks.

## 4. Ceph pool for Kubernetes RBD

Check the cluster and pool list:

```bash
ceph -s
ceph osd pool ls detail
ceph fsid
```

Create the RBD pool. Choose a reasonable initial PG count; the PG autoscaler can adjust it:

```bash
ceph osd pool create "$CEPH_POOL" 8
ceph osd pool application enable "$CEPH_POOL" rbd
rbd pool init "$CEPH_POOL"
ceph osd pool ls detail
```

**Production:** verify desired `size` / `min_size`, replica placement, usable capacity and CRUSH rule before writing data. Example for a properly sized multi-host cluster:

```bash
# Only when at least three suitable OSD failure domains exist:
# ceph osd pool set "$CEPH_POOL" size 3
# ceph osd pool set "$CEPH_POOL" min_size 2
```

**Single-OSD lab only:**

```bash
# Not for production
# ceph osd pool set "$CEPH_POOL" size 1
# ceph osd pool set "$CEPH_POOL" min_size 1
```

On a single-OSD cluster, `POOL_NO_REDUNDANCY` is a **valid warning**; don't suppress it to imply production readiness.

## 5. Create least-privilege CephX credentials for CSI

```bash
ceph auth get-or-create "client.${CSI_USER}" \
  mon 'profile rbd' \
  osd "profile rbd pool=${CEPH_POOL}" \
  mgr "profile rbd pool=${CEPH_POOL}"
```

The command **prints the key**. Avoid putting its output into CI logs, terminals being recorded, or Git. A safer pattern is to write a root-only local key file:

```bash
umask 077
ceph auth get-key "client.${CSI_USER}" > /root/ceph-csi-userKey
chmod 600 /root/ceph-csi-userKey
```

If a key was exposed, **rotate it** using a procedure supported by your exact Ceph release, then update dependent Secrets atomically. Never use `client.admin` as the CSI identity.

## 6. Prepare Kubernetes nodes

All Kubernetes nodes that may mount a volume need:

- Network reachability to Ceph MONs (modern messenger TCP **3300**, legacy **6789** where enabled) and the **actual OSD listening ports**, typically within the configured OSD service range.
- Supported Linux kernel RBD (`krbd`), tools and mount privileges; verify node plugin/host mount paths.
- An operational image registry path. With an **online** cluster, images can be pulled from Quay and `registry.k8s.io`; an internal mirror is optional.

Example checks from every Kubernetes node:

```bash
nc -vz -w 3 "$CEPH_MON_IP" 3300
nc -vz -w 3 "$CEPH_MON_IP" 6789  # only if enabled
lsmod | grep '^rbd' || true
```

From Ceph, check advertised endpoints and OSD address:

```bash
ceph mon dump
ceph osd metadata 0
```

Ceph CSI controller Pods run **inside the Kubernetes Pod network**, so validate Pod-to-Ceph connectivity as well as host-to-Ceph connectivity. Ensure Cilium/NetworkPolicy and firewalls permit the traffic.

## 7. Obtain a pinned Ceph CSI Helm chart on an online host

> **Version caution:** The verified source `v3.18.1` has `charts/ceph-csi-rbd/Chart.yaml` with chart version `3-canary` and appVersion `canary`. **Do not assume** a packaged `ceph-csi-rbd-3.18.1.tgz` exists at `https://ceph.github.io/csi-charts/ceph-csi-rbd-3.18.1.tgz` — that guessed URL returned 404. Build from a tagged source checkout and override the package metadata deliberately.

```bash
apt-get install -y git
mkdir -p /opt/ceph-csi-install && cd /opt/ceph-csi-install
git clone https://github.com/ceph/ceph-csi.git
cd ceph-csi
git checkout v3.18.1
cd ..
helm package ./ceph-csi/charts/ceph-csi-rbd \
  --version 3.18.1 --app-version v3.18.1 --destination .
helm show chart ./ceph-csi-rbd-3.18.1.tgz
```

This packages **the tagged source** with reproducible chart metadata; it does not make the chart officially released under that packaging version. Review the repository's current supported deployment method (the project has deprecated Helm charts in favor of the Operator). Version `v3.18.1` was used in the lab, but **Kubernetes 1.37.1 compatibility requires functional verification**.

## 8. Online Helm values for Ceph CSI RBD

Create `ceph-csi-online-values.yaml`:

```yaml
csiConfig:
  - clusterID: "REPLACE_WITH_CEPH_FSID"
    monitors:
      - "REPLACE_WITH_MON_IP:3300"

logLevel: 2
sidecarLogLevel: 1

nodeplugin:
  plugin:
    image:
      repository: quay.io/cephcsi/cephcsi
      tag: v3.18.1
  registrar:
    image:
      repository: registry.k8s.io/sig-storage/csi-node-driver-registrar
      tag: v2.18.0

provisioner:
  replicaCount: 2
  plugin:
    image:
      repository: quay.io/cephcsi/cephcsi
      tag: v3.18.1
  provisioner:
    image:
      repository: registry.k8s.io/sig-storage/csi-provisioner
      tag: v6.3.0
  attacher:
    image:
      repository: registry.k8s.io/sig-storage/csi-attacher
      tag: v4.13.0
  resizer:
    image:
      repository: registry.k8s.io/sig-storage/csi-resizer
      tag: v2.2.1
  snapshotter:
    image:
      repository: registry.k8s.io/sig-storage/csi-snapshotter
      tag: v8.6.0

storageClass:
  create: false
secret:
  create: false
volumeSnapshotClass:
  create: false
volumeGroupSnapshotClass:
  create: false
```

Replace the FSID and MON address before use:

```bash
sed -i 's/REPLACE_WITH_CEPH_FSID/47716fd6-db2e-11f1-a7eb-005056995454/' ceph-csi-online-values.yaml
sed -i 's/REPLACE_WITH_MON_IP/192.168.72.50/' ceph-csi-online-values.yaml
```

In a new environment use **your** FSID and MON list, not the reference values above. For production specify all reachable MON endpoints.

Render and verify image locations before installing:

```bash
helm lint ./ceph-csi-rbd-3.18.1.tgz -f ceph-csi-online-values.yaml
helm template ceph-csi-rbd ./ceph-csi-rbd-3.18.1.tgz \
  --namespace ceph-csi -f ceph-csi-online-values.yaml > rendered.yaml
grep -E '^[[:space:]]*image:' rendered.yaml | sort -u
```

If using a private registry instead, replace each repository name with the **actual tested internal path**. On containerd 2.x, an HTTP-only internal registry requires a correctly configured `hosts.toml` / CRI registry `config_path` **on every relevant Kubernetes node**. Prefer an HTTPS registry with trusted CA for production.

## 9. Install and validate Ceph CSI

```bash
helm upgrade --install ceph-csi-rbd ./ceph-csi-rbd-3.18.1.tgz \
  --namespace ceph-csi --create-namespace \
  -f ceph-csi-online-values.yaml --wait --timeout 10m

kubectl get pods -n ceph-csi -o wide
kubectl get deployment,daemonset -n ceph-csi
kubectl get csidrivers
helm status ceph-csi-rbd -n ceph-csi
```

Do **not** proceed to real workloads if the controller or node plugin isn't healthy. If snapshotting is needed, install/verify the Kubernetes VolumeSnapshot CRDs and snapshot-controller separately; the chart's sidecar image alone is not enough.

## 10. Create the CephX Kubernetes Secret securely

Copy the root-only key file to the Kubernetes admin host over a secure channel. Ensure the `ceph-csi` Namespace exists, then:

```bash
kubectl -n ceph-csi create secret generic csi-rbd-secret \
  --from-literal=userID=kubernetes-rbd \
  --from-file=userKey=/root/ceph-csi-userKey
```

The `userID` omits the `client.` prefix. Ensure the file includes the **key only**, not a full keyring block. For GitOps, use an approved encrypted Secret manager, External Secrets, Sealed Secrets or SOPS; **do not commit a plaintext `Secret` YAML**.

## 11. Define an RBD StorageClass

Create `storageclass-ceph-rbd.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: "47716fd6-db2e-11f1-a7eb-005056995454"
  pool: kubernetes-rbd
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi
  csi.storage.k8s.io/controller-publish-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: ceph-csi
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

Replace the example FSID and pool as appropriate. Apply:

```bash
kubectl apply -f storageclass-ceph-rbd.yaml
kubectl get storageclass
```

For production, assess whether `reclaimPolicy: Retain` is preferable for critical persistent data; `Delete` removes the provisioned backing RBD image after PVC/PV lifecycle deletion.

## 12. End-to-end PVC and Pod mount test

Create `rbd-smoke-test.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-rbd-smoke
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: ceph-rbd
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ceph-rbd-smoke
spec:
  restartPolicy: Never
  containers:
    - name: tester
      image: busybox:1.36.1
      command: ["sh", "-c", "echo ceph-csi-ok > /data/health.txt; cat /data/health.txt; sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ceph-rbd-smoke
```

```bash
kubectl apply -f rbd-smoke-test.yaml
kubectl get pvc,pod -w
kubectl exec ceph-rbd-smoke -- cat /data/health.txt
kubectl describe pvc ceph-rbd-smoke
kubectl get pv
```

Success = PVC `Bound`, Pod `Running`, and `ceph-csi-ok` returned from `/data/health.txt`. Also inspect Ceph:

```bash
rbd ls -p kubernetes-rbd
ceph -s
```

Clean up **test resources only** (with `reclaimPolicy: Delete`, this deletes the backing image):

```bash
kubectl delete -f rbd-smoke-test.yaml
```

## 13. Troubleshooting

| Symptom | Checks |
|---|---|
| `HTTP response to HTTPS client` while pulling a private image | Registry is HTTP but containerd assumes HTTPS; configure trusted HTTPS or a per-registry `hosts.toml` and verify CRI `config_path` on every node. |
| `ImagePullBackOff` | `kubectl describe pod -n ceph-csi ...`, inspect exact image path/tag, registry authentication and node reachability. |
| PVC stuck `Pending` | CSI provisioner logs, CephX caps, pool application `rbd`, MON connectivity, CSI FSID in `csiConfig` / StorageClass. |
| Pod stuck `ContainerCreating` / RBD mount fails | CSI node plugin logs, OSD port accessibility, kernel RBD support, node mount privileges and SELinux/AppArmor. |
| Ceph `TOO_FEW_OSDS` | Inspect actual OSD count and effective MON defaults; avoid assuming an old health warning is harmless. |
| Ceph `POOL_NO_REDUNDANCY` | Pool `size=1` is not replicated; add failure domains and increase replication rather than muting the warning. |
| Browser shows HTTPS warning on Dashboard | Replace the bootstrap self-signed certificate with a CA-signed certificate containing the correct DNS/IP SAN; trust that CA on admin clients. |
| OSD missing after reboot in sparse-file lab | Inspect `losetup`, LVM VG activation and `ceph-<FSID>@osd.0` systemd drop-in; do not recreate/wipe the existing PV/LV/OSD. |

Useful commands:

```bash
ceph -s
ceph health detail
ceph orch ps --refresh
ceph osd pool ls detail
kubectl get pods -n ceph-csi -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl get pvc,pv,storageclass
```

## 14. Production readiness checklist

- [ ] At least three appropriate Ceph failure domains and MON quorum; capacity and CRUSH rule validated.
- [ ] Multiple OSDs on **dedicated physical/virtual block devices**; no root-filesystem-backed sparse production OSDs.
- [ ] Replicated pool, failure/recovery testing, monitoring/alerting and backup strategy.
- [ ] Correct NTP, firewall and NetworkPolicy access from CSI controller Pods and Worker nodes to MONs and OSDs.
- [ ] Reproducible pinned Ceph and CSI versions; Kubernetes version compatibility tested.
- [ ] Image provenance/signature or digest policy, appropriate registry TLS and per-node image access.
- [ ] Rotated and least-privilege CephX credentials; secret storage/RBAC reviewed.
- [ ] CA-signed Ceph Dashboard TLS and restricted admin access.
- [ ] PVC provisioning, actual mount, restart, expansion, deletion and recovery tests completed.
- [ ] Confirm backup/restore and multi-node failover before critical applications such as Nexus.

## 15. Reference links

1. Ceph Squid cephadm installation: https://docs.ceph.com/en/squid/cephadm/install/
2. Ceph RBD with CSI: https://docs.ceph.com/en/latest/csi/rbd/
3. Ceph CSI pinned tag: https://github.com/ceph/ceph-csi/tree/v3.18.1
4. Pinned Helm chart defaults: https://github.com/ceph/ceph-csi/blob/v3.18.1/charts/ceph-csi-rbd/values.yaml
5. Pinned Helm metadata: https://github.com/ceph/ceph-csi/blob/v3.18.1/charts/ceph-csi-rbd/Chart.yaml
6. Containerd registry hosts configuration: https://github.com/containerd/containerd/blob/main/docs/hosts.md
7. Ceph Dashboard SSL: https://docs.ceph.com/en/squid/mgr/dashboard/

---

**License/note:** This runbook is newly written operational documentation. Adjust all addresses, cluster IDs, security settings and version pins to the target environment before executing it. The sample single-host/loop-backed configuration is for **education only**.
