# "x509: certificate has expired or is not yet valid" in Kubernetes

**Don't Panic.** The cluster turned one year old and nobody brought cake — or
certificates. `kubectl` may stop working entirely, but workloads are often still
running. Work out *which* certificate expired, then renew the way your
distribution expects.

> Longer walkthrough (planned): https://agency42.se/en/dont-panic/x509-certificate-has-expired

## What you see

```text
Unable to connect to the server: x509: certificate has expired or is not yet valid: current time … is after …
Unable to connect to the server: tls: failed to verify certificate: x509: certificate has expired or is not yet valid
Internal error occurred: failed calling webhook "…": … x509: certificate has expired or is not yet valid
```

Nodes may go `NotReady`; API server / etcd logs may show `tls: bad certificate`.

## What's going on

Kubernetes is held together by mutual TLS: API server, etcd, controller-manager,
scheduler, kubelets, admission webhooks, and your kubeconfig. On kubeadm and
RKE2, leaf certificates last about **one year**, CAs about **ten**. kubeadm
renews on control-plane upgrades; RKE2 renews on service start. Clusters that
"just work" and never get upgraded or restarted are the ones that break.

Read the message carefully:

- **"current time … is after …"** → certificate really expired.
- **"current time … is before …"** → clock skew / NTP. Fix time **before**
  touching certificates.

## Confirm it

| Where the error shows up | Likely culprit |
|---|---|
| `kubectl`: Unable to connect to the server | API server serving cert, or client cert in kubeconfig |
| failed calling webhook … | That webhook's serving cert |
| Node `NotReady`, kubelet client-cert complaints | Kubelet client certificate |
| etcd: `tls: bad certificate` | etcd peer/server certificates |

```bash
# API server serving certificate
echo | openssl s_client -connect api.cluster.example:6443 2>/dev/null \
  | openssl x509 -noout -subject -enddate

# Client cert in current kubeconfig (if present)
kubectl config view --raw --minify \
  -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d \
  | openssl x509 -noout -subject -enddate

# Kubelet client cert (on the node)
sudo openssl x509 -noout -enddate -in /var/lib/kubelet/pki/kubelet-client-current.pem

# Clock sanity (every node)
timedatectl
chronyc tracking
```

Distribution helpers (on a control-plane node):

```bash
sudo kubeadm certs check-expiration          # kubeadm
sudo rke2 certificate check --output table   # RKE2 (--output needs newer releases)
```

## Fix

### kubeadm

On **each** control-plane node (one at a time on HA):

```bash
# 0. Backup
sudo cp -a /etc/kubernetes /root/etc-kubernetes-backup-$(date +%F)

# 1. Renew all kubeadm-managed certs from the on-disk CA
sudo kubeadm certs renew all

# 2. Restart static control-plane pods (kubectl can't do this).
#    Move manifests out, wait until stopped, move back.
sudo mkdir -p /root/manifests-parked
sudo mv /etc/kubernetes/manifests/*.yaml /root/manifests-parked/
while sudo crictl ps --name 'kube-apiserver|kube-controller-manager|kube-scheduler|etcd' -q | grep -q .; do sleep 5; done
sudo mv /root/manifests-parked/*.yaml /etc/kubernetes/manifests/

# 3. Refresh admin kubeconfig and verify
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
sudo kubeadm certs check-expiration
kubectl get nodes
```

`kubeadm certs renew` can use existing certs as the SAN source even while the
API is down (it may warn it can't read cluster config). It does **not** renew
the CA — a 10-year CA expiry is a planned rotation, not a mid-incident fix.
Kubelet client certs live under `/var/lib/kubelet/pki` and rotate separately;
see kubeadm docs if a node was offline long enough for those to expire too.

### RKE2

> Running RKE2 / Rancher? See also [agency42.se/rancher](https://agency42.se/rancher).

RKE2 renews client/server certs that are expired or within **120 days** of
expiry **on every start**:

```bash
# Servers one at a time, then agents
sudo systemctl restart rke2-server
sudo systemctl restart rke2-agent

sudo rke2 certificate check --output table
```

Brand-new keys instead of extending existing ones:

```bash
sudo systemctl stop rke2-server
sudo rke2 certificate rotate          # or: --service api-server,kubelet,...
sudo systemctl start rke2-server
```

RKE2 rewrites `/etc/rancher/rke2/rke2.yaml` on server start — recopy your
kubeconfig if you keep a copy elsewhere. Downstream Rancher-managed clusters:
use **Rotate Certificates** in Cluster Management rather than node-by-node.

### OpenShift 4.x

Operators rotate continuously; expiry usually follows a long shutdown. Control
plane often recovers on its own; **approve pending kubelet CSRs**:

```bash
oc get csr
oc describe csr <csr_name>              # confirm legitimate requestor
oc adm certificate approve <csr_name>
```

Approve `node-bootstrapper` CSRs first; on user-provisioned infra, also
`kubelet-serving` that follow. Don't blindly approve every CSR in the cluster.

### Webhook certificates only

If only webhook calls fail, cluster PKI may be fine. With cert-manager:

```bash
kubectl get certificate -A
kubectl describe certificate <name> -n <ns>
```

A webhook with `failurePolicy: Fail` can block all deploys — fix the issuer
before deleting webhook configs in a panic.

## Prevent

- Upgrade at least once a year (kubeadm renews on CP upgrade; RKE2 on restart).
- Monitor expiry: scheduled `kubeadm certs check-expiration`, probe `:6443`
  (`probe_ssl_earliest_cert_expiry`), RKE2
  `kubectl get events -A --field-selector reason=CertificateExpirationWarning`.
- NTP on every node (chrony / systemd-timesyncd).
- Calendar the 10-year CA expiry.
- cert-manager for app/webhook certs; alert on Certificates that aren't Ready.

Before a risky upgrade, run companion checks:
[k8s-preflight](https://github.com/Agency-42/k8s-preflight).

## Notes for OpenShift / RKE2

| | kubeadm | RKE2 | OpenShift 4.x |
|---|---|---|---|
| Leaf lifetime | ~1 year | 365 days | Short-lived, auto-rotated |
| Auto renewal | On `kubeadm upgrade` | On service start (≤120 days left) | Continuous (operators) |
| Manual fix | `kubeadm certs renew all` + static pod restart | Restart `rke2-server`/`agent` or `rke2 certificate rotate` | Approve pending CSRs |
| Check | `kubeadm certs check-expiration` | `rke2 certificate check` | `oc get csr`, clusteroperators |

## Related

- [The towel](../towel.md)
- [k8s-preflight](https://github.com/Agency-42/k8s-preflight) — pre-upgrade certs and skew checks
- Agency 42 · [Kubernetes](https://agency42.se/kubernetes) · [Rancher / RKE2](https://agency42.se/rancher) · [agency42.se](https://agency42.se)
