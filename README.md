
# NDC Kubernetes HA Cluster Rebuild

**Manual Execution Runbook | F5 VIP | Kubernetes v1.36.5**

**Date:** 24 October 2026

## Overview

This runbook describes how to rebuild the existing 11-node Kubernetes HA cluster:

1. Drain and remove all worker nodes.
2. Reset all eight workers.
3. Reset the secondary control-plane nodes.
4. Reset the primary control-plane node.
5. Initialize Kubernetes through the F5 VIP.
6. Install Calico CNI.
7. Join the remaining two control-plane nodes.
8. Join all eight workers.
9. Verify cluster health and F5 failover.

> **Warning:** This procedure permanently destroys existing Kubernetes cluster state. No backup procedure is included. Expect service downtime.

## 1. Target Configuration

| Component | Configuration |
|---|---|
| Kubernetes | v1.36.5 |
| F5 VIP | `172.31.150.60:6443` |
| Primary master | `ndc-master-01` |
| Additional masters | `ndc-master-02`, `ndc-master-03` |
| Pod CIDR | `10.224.0.0/16` |
| Service CIDR | `10.96.0.0/24` |
| CNI | Calico |
| SSH username | `tbl` |
| Administrative access | `sudo` |

### Worker Nodes

| Node | Role |
|---|---|
| ndc-worker-core-01 | Worker |
| ndc-worker-core-02 | Worker |
| ndc-worker-dist-01 | Worker |
| ndc-worker-dist-02 | Worker |
| ndc-worker-dmz-01 | Worker |
| ndc-worker-dmz-02 | Worker |
| ndc-worker-ext-01 | Worker |
| ndc-worker-ext-02 | Worker |

## 2. Preflight Checks

Run from `ndc-master-01`:

```bash
whoami
sudo -v
kubectl get nodes -o wide
kubectl get pods -A
sudo kubeadm version
sudo systemctl is-active containerd
```

### Verify SSH Access

```bash
ssh tbl@ndc-master-02 "sudo -v; hostname"
ssh tbl@ndc-master-03 "sudo -v; hostname"
```

Repeat for all workers. If sudo requires a password, execute commands interactively.

### Verify F5 VIP

Confirm with the F5 administrator:

- VIP: `172.31.150.60:6443`
- Protocol: TCP Layer 4 passthrough
- Backend: all three master nodes on TCP 6443
- Suitable health monitors
- `ndc-master-01` can become healthy independently during initialization

```bash
getent hosts ndc-master-01 ndc-master-02 ndc-master-03
nc -vz -w 3 172.31.150.60 6443
```

Before the API server starts, TCP connection failure may be expected.

## 3. Drain Worker Nodes

Execute on `ndc-master-01`.

```bash
WORKERS=(
  ndc-worker-core-01
  ndc-worker-core-02
  ndc-worker-dist-01
  ndc-worker-dist-02
  ndc-worker-dmz-01
  ndc-worker-dmz-02
  ndc-worker-ext-01
  ndc-worker-ext-02
)

for node in "${WORKERS[@]}"; do
  echo "Draining $node"

  kubectl drain "$node" \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --timeout=15m || break
done
```

> Stop and investigate if any drain fails. The `--delete-emptydir-data` option permits deletion of temporary pod data.

After **all eight nodes have drained successfully**, delete their Kubernetes objects:

```bash
for node in "${WORKERS[@]}"; do
  kubectl delete node "$node"
done
```

## 4. Reset Worker Nodes

Execute the following commands individually on all eight workers as `tbl`.

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni

sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Do not blindly flush firewall or iptables rules.

## 5. Reset Secondary Master Nodes

Reset `ndc-master-03` first.

```bash
ssh tbl@ndc-master-03
```

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes

sudo systemctl restart containerd kubelet
```

Next, reset `ndc-master-02`.

```bash
ssh tbl@ndc-master-02
```

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes

sudo systemctl restart containerd kubelet
```

On `ndc-master-01`, remove the old Kubernetes node objects if they still exist:

```bash
kubectl delete node ndc-master-03 --ignore-not-found
kubectl delete node ndc-master-02 --ignore-not-found
```

## 6. Reset Primary Master

Execute on `ndc-master-01`.

```bash
sudo kubeadm reset -f

sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes

rm -f ~/.kube/config

sudo systemctl restart containerd kubelet
```

Verify that the old control-plane state is gone:

```bash
sudo ls -la /etc/kubernetes/manifests /var/lib/etcd \
  2>/dev/null || true

sudo ss -lntp | grep ":6443 " || true
```

## 7. Initialize Primary Master

Execute on `ndc-master-01`.

```bash
sudo kubeadm init \
  --kubernetes-version=v1.36.5 \
  --control-plane-endpoint="172.31.150.60:6443" \
  --apiserver-cert-extra-sans="172.31.150.60" \
  --pod-network-cidr="10.224.0.0/16" \
  --service-cidr="10.96.0.0/24" \
  --upload-certs
```

> Ensure the F5 VIP forwards connections to `ndc-master-01` during initialization.

### Configure kubectl

```bash
mkdir -p "$HOME/.kube"

sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"

sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"

chmod 600 "$HOME/.kube/config"
```

Verify:

```bash
kubectl get nodes
kubectl cluster-info
```

Keep the generated joining credentials secure.

## 8. Install Calico CNI

Install Calico before joining additional masters.

### Install CRDs and Operator

```bash
kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml

kubectl create -f \
  https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
```

### Configure Calico IP Pool

Create `calico-custom-resources.yaml`:

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - cidr: 10.224.0.0/16
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
```

Apply:

```bash
kubectl apply -f calico-custom-resources.yaml
```

### Verify Calico

```bash
kubectl get pods -n tigera-operator

kubectl get pods -n calico-system -o wide

kubectl get tigerastatus

kubectl get nodes -o wide

kubectl get ippools.crd.projectcalico.org -o wide
```

Do not proceed if Calico is degraded or CNI networking is not operational.

## 9. Join Secondary Masters

Generate joining credentials on `ndc-master-01` if needed:

```bash
sudo kubeadm token create --print-join-command

sudo kubeadm init phase upload-certs --upload-certs
```

Run the following on `ndc-master-02` with the actual generated values:

```bash
sudo kubeadm join 172.31.150.60:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

Verify:

```bash
kubectl get nodes -o wide

kubectl -n kube-system get pods \
  -l component=etcd -o wide
```

After master 02 is healthy, repeat the join command on `ndc-master-03`.

## 10. Join Worker Nodes

Generate a worker joining command on `ndc-master-01`:

```bash
sudo kubeadm token create --print-join-command
```

Execute on each worker:

```bash
sudo kubeadm join 172.31.150.60:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

Do not include `--control-plane` when joining workers.

## 11. Cluster Health Checks

On `ndc-master-01`:

```bash
kubectl get nodes -o wide

kubectl get pods -A -o wide

kubectl get tigerastatus

kubectl -n kube-system get pods \
  -l component=etcd -o wide

kubectl cluster-info
```

Expected state:

- Three control-plane nodes: Ready
- Eight worker nodes: Ready
- Calico components: Healthy
- CoreDNS: Running
- F5 backend members: Healthy

### Verify Service Network

```bash
kubectl get svc -A

kubectl -n kube-system get cm kubeadm-config \
  -o jsonpath='{.data.ClusterConfiguration}'
```

Confirm:

```yaml
networking:
  podSubnet: 10.224.0.0/16
  serviceSubnet: 10.96.0.0/24
```

## 12. F5 Failover Verification

Verify the Kubernetes API endpoint:

```bash
kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}'
```

Expected:

```text
https://172.31.150.60:6443
```

Coordinate with the F5 administrator to disable one healthy backend pool member at a time.

Verify Kubernetes API access continues through the VIP:

```bash
kubectl get nodes
kubectl cluster-info
```

Re-enable each backend after the test.

Do not remove more than one stacked-etcd control-plane member at a time during the validation.

## 13. Troubleshooting

| Problem | Recommended check |
|---|---|
| kubeadm init timeout | F5 pool, TCP 6443, API startup |
| Unstable controlPlaneEndpoint | kubeadm-config ConfigMap |
| Certificate key expired | Re-upload certificates |
| Worker NotReady | kubelet, containerd, Calico |
| Calico degraded | Operator pods, IPPool, CNI settings |
| Pod networking failure | VXLAN UDP 4789, routing, firewall |
| F5 backend unavailable | Pool health monitor and backend routing |

### Regenerate Certificate Upload Key

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

### Regenerate Join Token

```bash
sudo kubeadm token create --print-join-command
```

## 14. Final Checklist

- [ ] Eight worker nodes drained successfully
- [ ] Worker machines reset
- [ ] Secondary masters reset
- [ ] Primary master reset
- [ ] F5 VIP configured correctly
- [ ] Primary master initialized
- [ ] Pod CIDR configured as `10.224.0.0/16`
- [ ] Service CIDR configured as `10.96.0.0/24`
- [ ] Calico installed and healthy
- [ ] Master 02 joined successfully
- [ ] Master 03 joined successfully
- [ ] Eight workers joined successfully
- [ ] All 11 nodes Ready
- [ ] CoreDNS healthy
- [ ] F5 failover verified

## References

- [Kubernetes HA with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [Calico Configuration](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/config-options)
- [Calico Requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)
