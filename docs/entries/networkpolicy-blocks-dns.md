# Default-deny NetworkPolicy broke DNS: pods can't resolve anything

**Don't Panic.** You applied a responsible default-deny NetworkPolicy, and a few
seconds later every service forgot how to find anything. NetworkPolicy is doing
exactly what you told it to — including blocking DNS.

> Longer walkthrough (planned): https://agency42.se/en/dont-panic/networkpolicy-blocks-dns

## What you see

Right after `kubectl apply` of a NetworkPolicy, a Helm upgrade with
`networkPolicy.enabled: true`, or a "baseline security" rollout:

```text
dial tcp: lookup postgres.my-app.svc.cluster.local on 10.96.0.10:53: read udp … i/o timeout
;; connection timed out; no servers could be reached
curl: (6) Could not resolve host: api.example.com
getaddrinfo EAI_AGAIN api.example.com
java.net.UnknownHostException: postgres
```

Readiness probes fail; apps that resolve a hostname at startup go
`CrashLoopBackOff`. The database is usually fine — pods just can't look it up.

## What's going on

NetworkPolicies are allow-lists. Once **any** policy selects a pod and lists
`Egress` in `policyTypes`, every outbound connection is denied unless some
policy explicitly allows it. DNS is outbound traffic (UDP, or TCP for large
answers). Kubernetes docs state it: a default deny-all egress also blocks DNS.

Two traps:

1. Pods query the DNS **Service** IP, but most CNIs evaluate egress **after**
   DNAT to the CoreDNS **pod** IP and **target port**. Match the DNS pods and
   the port those pods listen on — not only the Service port.
2. CoreDNS is not in your namespace. A bare `podSelector` only matches pods in
   the policy's own namespace. You need `namespaceSelector` **and**
   `podSelector` in the **same** list item (AND).

## Confirm it

Replace `my-app` with your namespace.

```bash
# 1. Policies in the namespace — any listing Egress?
kubectl get networkpolicy -n my-app
kubectl describe networkpolicy -n my-app

# 2. Reproduce from inside the affected namespace
kubectl -n my-app run dns-test --rm -it --restart=Never \
  --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local

# 3. Control group: fresh namespace, no policies
kubectl create namespace dns-check
kubectl -n dns-check run dns-test --rm -it --restart=Never \
  --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local
kubectl delete namespace dns-check
```

If step 2 times out and step 3 answers → it's your NetworkPolicy. If both fail,
look at CoreDNS itself.

Where DNS lives and which port the pods use:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide --show-labels

kubectl get endpointslices -n kube-system -l kubernetes.io/service-name=kube-dns \
  -o jsonpath='{range .items[*]}{.ports}{"\n"}{end}'

kubectl get namespace kube-system --show-labels
# expect: kubernetes.io/metadata.name=kube-system

# NodeLocal DNSCache?
kubectl get daemonset -n kube-system node-local-dns
kubectl -n my-app exec deploy/my-app -- cat /etc/resolv.conf
```

If `/etc/resolv.conf` shows `nameserver 169.254.20.10`, pods talk to a
node-local cache — see the NodeLocal note under Fix.

## Fix

Allow DNS egress for every pod in the namespace, then (re)apply the deny.
**Apply the DNS rule first** so there is no window without DNS.

Ready-made manifests:
[`manifests/networkpolicy/`](../../manifests/networkpolicy/)
(vanilla + OpenShift). Same pair lives in
[kubernetes-best-practices](https://github.com/Agency-42/kubernetes-best-practices/tree/main/examples/network-policies).

Vanilla Kubernetes / RKE2:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: my-app
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f 01-allow-dns-egress.yaml
kubectl apply -f 00-default-deny.yaml
```

### Ways this "fix" still fails

**1. One dash too many.** `namespaceSelector` and `podSelector` in the *same*
list item = AND. Drop `namespaceSelector` (or put `podSelector` in its own
item) and you only match `kube-dns` pods **in my-app** — i.e. nothing:

```yaml
  egress:
    - to:
        - podSelector:          # no namespaceSelector → wrong namespace
            matchLabels:
              k8s-app: kube-dns
```

**2. UDP only.** Large answers fall back to TCP. Allow both.

**3. NodeLocal DNSCache.** Host-network cache won't match a pod selector. Add:

```yaml
    - to:
        - ipBlock:
            cidr: 169.254.20.10/32
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Re-run the `dns-test` pod. You should get an answer; everything else stays
blocked until you add explicit allows.

## Prevent

- Ship deny + DNS allow as a pair (same Helm chart / Kustomize / GitOps folder).
- Pipeline smoke test: `nslookup` Job right after policies apply in staging.
- Cluster-wide DNS baseline where your CNI supports it (e.g. Calico
  `GlobalNetworkPolicy`, Cilium clusterwide policy, `AdminNetworkPolicy`).
- Remember: plain Flannel ignores NetworkPolicy — "worked in dev" often means
  that.

## Notes for OpenShift / RKE2

**RKE2:** Canal enforces NetworkPolicy. CoreDNS in `kube-system`,
`k8s-app=kube-dns`, port 53 — vanilla manifest works. Default DNS Service is
often `10.43.0.10`. NodeLocal via `rke2-coredns` → add the `ipBlock` rule.
CIS profile may add `default-network-dns-policy` in `kube-system`; keep it if
you tighten that namespace.

**OpenShift 4.x:** DNS pods live in **`openshift-dns`**, label
`dns.operator.openshift.io/daemonset-dns=default`. Service on 53, **pods on
5353**. Policy matches the pod port — allowing only 53 still breaks DNS.

```bash
oc get pods -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default
oc get endpointslices -n openshift-dns -l kubernetes.io/service-name=dns-default \
  -o jsonpath='{range .items[*]}{.ports}{"\n"}{end}'
```

Use
[`01-allow-dns-egress-openshift.yaml`](../../manifests/networkpolicy/01-allow-dns-egress-openshift.yaml)
(`openshift-dns`, that label, UDP+TCP **5353**).

## Related

- [The towel](../towel.md) — quick DNS triage
- [OpenShift SCC admission](openshift-unable-to-validate-against-any-scc.md)
- Agency 42 · [Kubernetes](https://agency42.se/kubernetes) · [DevOps & SecOps / network security](https://agency42.se/devops-secops) · [Rancher / RKE2](https://agency42.se/rancher)
- [kubernetes-best-practices network-policies](https://github.com/Agency-42/kubernetes-best-practices/tree/main/examples/network-policies)
- Agency 42 · [agency42.se](https://agency42.se)
