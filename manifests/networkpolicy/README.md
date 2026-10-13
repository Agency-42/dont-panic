# NetworkPolicy examples: default deny without breaking DNS

A default-deny policy is a good baseline. A default-deny policy *without* a DNS
exception is a very fast way to take down every pod in a namespace.
These examples come as a pair.

| File | Use on |
|---|---|
| `00-default-deny.yaml` | All distributions |
| `01-allow-dns-egress.yaml` | Vanilla Kubernetes (kubeadm, EKS, AKS, GKE), RKE2 |
| `01-allow-dns-egress-openshift.yaml` | OpenShift 4.x (DNS pods listen on **5353**) |

Also shipping in [`kubernetes-best-practices`](https://github.com/Agency-42/kubernetes-best-practices/tree/main/examples/network-policies) for preventive use.

## Apply

Change `namespace: my-app` to your namespace, then apply the DNS rule **first**
so there is no window without DNS:

```bash
kubectl apply -f 01-allow-dns-egress.yaml      # or the -openshift variant
kubectl apply -f 00-default-deny.yaml
```

## Verify

```bash
kubectl -n my-app run dns-test --rm -it --restart=Never \
  --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local
```

You should get an answer. External traffic and traffic between pods are still
blocked. That's the point: now add explicit allow rules for what each workload
actually needs.

## Notes

- NetworkPolicies are additive. Allow rules add up; nothing "overrides" the deny.
- Your CNI must enforce NetworkPolicy. Flannel alone does not — these manifests
  are then silently ignored.
- OpenShift: the Service listens on 53, but pods listen on **5353**. Policy is
  evaluated against the pod port. See
  [docs/entries/networkpolicy-blocks-dns.md](../../docs/entries/networkpolicy-blocks-dns.md).
