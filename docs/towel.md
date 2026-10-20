# The towel: one-page Kubernetes triage cheat sheet

> *Always know where your towel is.*

Copy-pasteable first moves when something is on fire. Vanilla Kubernetes and
RKE2 use `kubectl`. OpenShift: prefer `oc` (same flags work on most of these).

## Cluster pulse

```bash
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl get events -A --sort-by='.lastTimestamp' | tail -40
kubectl top nodes 2>/dev/null; kubectl top pods -A 2>/dev/null | head
```

## Pod won't start / keeps restarting

```bash
kubectl describe pod <pod> -n <ns>          # Events at the bottom
kubectl get pod <pod> -n <ns> -o yaml | less
kubectl logs <pod> -n <ns> --previous       # last crashed container
kubectl logs <pod> -n <ns> -c <container>
kubectl get events -n <ns> --field-selector involvedObject.name=<pod>
```

Exit codes worth knowing: **137** = OOMKilled (or SIGKILL), **139** = segfault,
**1** / app-specific = process exited. `CrashLoopBackOff` is a symptom, not a cause.

See [crashloopbackoff](entries/crashloopbackoff.md).

## DNS / "could not resolve host"

```bash
kubectl get networkpolicy -A
kubectl -n <ns> run dns-test --rm -it --restart=Never \
  --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide --show-labels
# OpenShift DNS:
oc get pods -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default
```

See [networkpolicy-blocks-dns](entries/networkpolicy-blocks-dns.md).

## Certificates / "x509: certificate has expired"

```bash
echo | openssl s_client -connect <api-host>:6443 2>/dev/null \
  | openssl x509 -noout -subject -enddate
kubectl config view --raw --minify \
  -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d \
  | openssl x509 -noout -subject -enddate
sudo kubeadm certs check-expiration          # kubeadm
sudo rke2 certificate check --output table   # RKE2 (newer releases)
timedatectl; chronyc tracking                # "not yet valid" → clock first
```

See [x509-certificate-has-expired](entries/x509-certificate-has-expired.md).

## OpenShift: no pods, FailedCreate / SCC

```bash
oc get events -n <ns> --field-selector reason=FailedCreate
oc describe replicaset -n <ns>
oc get namespace <ns> \
  -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}{"\n"}'
oc get deployment <name> -n <ns> -o yaml | oc adm policy scc-subject-review -f -
```

See [openshift-unable-to-validate-against-any-scc](entries/openshift-unable-to-validate-against-any-scc.md).

## Network / "connection refused" vs timeout

```bash
kubectl get svc,endpoints,endpointslices -n <ns>
kubectl get networkpolicy -n <ns> -o wide
kubectl describe networkpolicy -n <ns>
# From a debug pod in the same namespace:
kubectl -n <ns> run netshoot --rm -it --restart=Never --image=nicolaka/netshoot -- bash
```

Timeout often means NetworkPolicy or no route. Connection refused often means
the process isn't listening (wrong port, CrashLoop, readiness).

## Control plane / API unreachable

```bash
# On a control-plane node (kubeadm static pods):
sudo crictl ps
sudo journalctl -u kubelet -n 100 --no-pager
# RKE2:
sudo systemctl status rke2-server   # or rke2-agent
sudo journalctl -u rke2-server -n 100 --no-pager
# OpenShift:
oc get clusteroperators
oc get nodes
oc get csr
```

## Before you change anything

1. Note the exact error string (screenshot or copy).
2. Check whether it's one namespace or the whole cluster.
3. Prefer read-only commands first (`get`, `describe`, `logs`).
4. If you're about to renew certs or restart control-plane components: take a
   backup of `/etc/kubernetes` (kubeadm) or confirm you have console/SSH access
   that does not depend on the API.

---

Part of [Don't Panic](../README.md) · Agency 42 · [agency42.se](https://agency42.se)
