# OpenShift: "unable to validate against any security context constraint"

**Don't Panic.** The Helm chart installed, the Deployment exists, and there are
zero pods. OpenShift isn't broken — it's refusing to run the container the way
the manifest asks. The error lists exactly why. Fix it without the "just grant
`privileged`" shortcut.

> Longer walkthrough (planned): https://agency42.se/en/dont-panic/openshift-unable-to-validate-against-any-scc

> **Verification status:** commands and SCC behaviour in this entry are checked
> against the OpenShift docs, **not** run on a live OpenShift/OKD lab cluster yet.
> Test in a throwaway namespace first.

## What you see

`oc get deployment` shows `0/1` ready, `oc get pods` shows nothing, events show:

```text
Warning  FailedCreate  replicaset/my-app-…  Error creating: pods "my-app-…" is forbidden:
unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount,
provider restricted-v2: .spec.securityContext.fsGroup: Invalid value: []int64{1001}: 1001 is not an allowed group,
provider restricted-v2: .containers[0].runAsUser: Invalid value: 1001: must be in the ranges: [1000680000, 1000689999],
provider "nonroot-v2": Forbidden: not usable by user or serviceaccount,
… provider "privileged": Forbidden: not usable by user or serviceaccount]
```

Other common `restricted-v2` lines:

```text
.containers[0].securityContext.privileged: Invalid value: true: Privileged containers are not allowed
spec.volumes[0]: Invalid value: "hostPath": hostPath volumes are not allowed to be used
.containers[0].capabilities.add: Invalid value: "NET_ADMIN": capability may not be added
.containers[0].securityContext.allowPrivilegeEscalation: Invalid value: true: …
```

## What's going on

Security Context Constraints (SCCs) are OpenShift's admission policy for pods.
On create, OpenShift tries every SCC the **pod's service account** (or creating
user) may `use`. If none accepts the spec, the pod is rejected **before it
exists** — hence no pods, only `FailedCreate` on the ReplicaSet / StatefulSet /
DaemonSet.

How to read each line:

- **`Forbidden: not usable by user or serviceaccount`** — SA has no `use` on
  that SCC (noise, but lists what exists).
- **`provider restricted-v2: <field>: <reason>`** — your to-do list.
  `restricted-v2` is the default for authenticated users on OpenShift 4.11+
  (UID from the namespace range, drop caps except `NET_BIND_SERVICE`, no
  privilege escalation / host access, seccomp `runtime/default`).

Every namespace gets a UID range (`openshift.io/sa.scc.uid-range`). Charts that
hardcode `runAsUser: 1001` / `fsGroup: 1001` almost never fall inside it.

**Why `oc run` works but the Deployment doesn't:** cluster admins often have
`anyuid` and high priority. Controllers create pods as the **service account**,
which only has `restricted-v2`. Same YAML, different admitter.

Newer releases may also list `restricted-v3` (`restricted-v2` + Linux user
namespaces / `hostUsers: false`).

## Confirm it

```bash
# 1. Rejection events
oc get events -n my-app --field-selector reason=FailedCreate
oc describe replicaset -n my-app -l app=my-app

# 2. Service account on the pod template (empty = default)
oc get deployment my-app -n my-app \
  -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'

# 3. Namespace UID range
oc get namespace my-app \
  -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}{"\n"}'

# 4. Would any usable SCC admit this pod?
oc get deployment my-app -n my-app -o yaml | oc adm policy scc-subject-review -f -

# 5. Can this SA use a given SCC?
oc auth can-i use securitycontextconstraints/nonroot-v2 \
  --as=system:serviceaccount:my-app:my-app -n my-app
```

If `scc-subject-review` shows nothing under `ALLOWED BY`, no usable SCC accepts
the spec. Fix the fields named in the `restricted-v2` lines.

## Fix

Work down this list; stop at the first option that works.

### Option 1 (best): fit `restricted-v2`

Remove hardcoded identity; let OpenShift assign UID/GID:

```yaml
spec:
  template:
    spec:
      securityContext: {}            # no runAsUser / runAsGroup / fsGroup
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            capabilities:
              drop: ["ALL"]
```

In Helm, clear/disable chart `securityContext` values (many charts have an
OpenShift toggle).

Image must tolerate a random UID in group 0 — group-own writable paths and
`g=u`:

```dockerfile
RUN chgrp -R 0 /app /var/cache/app && chmod -R g=u /app /var/cache/app
USER 1001
```

Use a **numeric** `USER`. A name like `USER appuser` breaks `runAsNonRoot`
verification (`cannot verify user is non-root`).

### Option 2: fixed non-root UID / fsGroup required

Some vendor images insist on a specific UID (e.g. Grafana `472`). Use
**`nonroot-v2`** (any non-zero UID, any fsGroup, otherwise restricted). Grant to
a **dedicated** SA in that namespace only:

```bash
oc create serviceaccount my-app -n my-app
oc create rolebinding my-app-scc-nonroot-v2 -n my-app \
  --clusterrole=system:openshift:scc:nonroot-v2 \
  --serviceaccount=my-app:my-app
oc set serviceaccount deployment/my-app my-app -n my-app
```

Shorthand: `oc adm policy add-scc-to-user nonroot-v2 -z my-app -n my-app`.

Pin the SCC on the pod template and verify:

```yaml
spec:
  template:
    metadata:
      annotations:
        openshift.io/required-scc: nonroot-v2
```

```bash
oc get pod -n my-app -l app=my-app \
  -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.metadata.annotations.openshift\.io/scc}{"\n"}{end}'
```

### Option 3: really needs root or host access

`anyuid`, `hostnetwork-v2`, `hostmount-anyuid`, `privileged` exist for node
agents, CSI, some monitoring. Same pattern: dedicated SA, dedicated namespace,
document why. For ordinary apps, rebuilding the image (option 1) is usually
cheaper than living with root.

### What not to do

```bash
oc adm policy add-scc-to-user privileged -z default -n my-app   # please don't
```

That privileges **every** pod using `default` in the namespace — including
whatever lands there next year.

## Prevent

- CI: render the chart and `oc adm policy scc-subject-review -f rendered.yaml`
  against a throwaway OpenShift/OKD namespace.
- Build images for arbitrary UIDs by default (group 0, `g=u`, numeric `USER`).
- One SA per workload; grant SCCs with namespaced RoleBindings — never to
  `default`.
- Audit: `oc get rolebindings,clusterrolebindings -A -o wide | grep system:openshift:scc`

## Notes for OpenShift / RKE2

Vanilla Kubernetes and RKE2 have no SCCs. They use **Pod Security Admission**
(namespace labels like `pod-security.kubernetes.io/enforce=restricted`):

```text
Error creating: pods "my-app-…" is forbidden: violates PodSecurity "restricted:latest":
allowPrivilegeEscalation != false (container "app" must set securityContext.allowPrivilegeEscalation=false), …
```

```bash
kubectl get namespace my-app --show-labels
```

**RKE2 with the CIS profile** defaults cluster-wide to `restricted`
(`/etc/rancher/rke2/rke2-pss.yaml`, system namespaces exempt). Charts that work
elsewhere often fail immediately. Philosophy is the same: change the pod spec
first; relax a namespace only when the workload truly needs it.

An image that runs under OpenShift `restricted-v2` almost always passes PSA
`restricted` too.

## Related

- [NetworkPolicy broke DNS](networkpolicy-blocks-dns.md)
- [The towel](../towel.md)
- Agency 42 · [Kubernetes](https://agency42.se/kubernetes) · [Rancher / RKE2](https://agency42.se/rancher) · [DevOps & SecOps](https://agency42.se/devops-secops) · [agency42.se](https://agency42.se)
