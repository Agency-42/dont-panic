<p align="center">
  <img src="assets/agent-leman.png" alt="Agent Le'man holding a DON'T PANIC flag" width="320">
</p>

<p align="center">
  <strong>Don't Panic.</strong>
</p>

<h1 align="center">Don't Panic — Kubernetes troubleshooting field guide</h1>

<p align="center">
  Symptom → diagnosis → fix, for vanilla Kubernetes, OpenShift and RKE2.<br>
  Maintained by <a href="https://agency42.se">Agency 42</a> — Sweden's go-to Kubernetes specialists.
</p>

<p align="center">
  <a href="https://agency42.se">agency42.se</a> ·
  <a href="https://github.com/Agency-42">Agency-42 on GitHub</a>
</p>

---

> *"A Kubernetes cluster is mostly harmless. Mostly."*

Your pods are timing out on DNS, `kubectl` just said `x509: certificate has expired`,
or OpenShift refused every SCC and left you with zero pods. Take a breath. Grab
your towel. **Agent Le'man** (that's *A Gent Le'man*, in the black fedora) has
seen this before.

This repo is a **field guide**: short, copy-pasteable entries and runnable
manifests. Longer articles will live on the site at planned URLs under
`https://agency42.se/en/dont-panic/<slug>` (those pages are not all published yet).

## How to use

1. Find the entry that matches the error you're googling (`docs/entries/`).
2. Work the sections in order: **What you see → What's going on → Confirm it → Fix → Prevent**.
3. Check **Notes for OpenShift / RKE2** when your distribution differs.
4. Keep [docs/towel.md](docs/towel.md) open — one-page triage for first moves.
5. Apply manifests from [`manifests/`](manifests/) when an entry points you there.

Prefer read-only commands first. If you're about to renew control-plane certs or
restart static pods, make sure you still have SSH/console access that does not
depend on the API.

## Entries

| Entry | Symptom |
|---|---|
| [networkpolicy-blocks-dns](docs/entries/networkpolicy-blocks-dns.md) | Default-deny NetworkPolicy; DNS lookups time out (`i/o timeout` on `:53`) |
| [x509-certificate-has-expired](docs/entries/x509-certificate-has-expired.md) | `x509: certificate has expired or is not yet valid` |
| [openshift-unable-to-validate-against-any-scc](docs/entries/openshift-unable-to-validate-against-any-scc.md) | `unable to validate against any security context constraint` |
| [crashloopbackoff](docs/entries/crashloopbackoff.md) | `CrashLoopBackOff`, restart count climbing (exit codes, OOMKilled, liveness probes) |

More entries are coming (ImagePullBackOff, Pending / FailedScheduling,
OOMKilled, and friends). Issues and PRs that add real field notes are
welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). Humans review before merge.

## Companion repos

| Repo | Role |
|---|---|
| [`k8s-preflight`](https://github.com/Agency-42/k8s-preflight) | Read-only checks **before** a Kubernetes minor upgrade (skew, deprecated APIs, kubelets, etcd, PDBs, certs). |
| [`kubernetes-best-practices`](https://github.com/Agency-42/kubernetes-best-practices) | Preventive checklists and example manifests — including the DNS-safe default-deny NetworkPolicy pair also vendored under [`manifests/networkpolicy/`](manifests/networkpolicy/). |

Don't Panic is for when you're already on fire. Preflight and best-practices are
for making Friday quieter.

## Layout

```
docs/towel.md                 # one-page kubectl/oc triage cheat sheet
docs/entries/                 # field-guide entries (4 so far)
manifests/networkpolicy/      # default-deny + DNS egress (vanilla + OpenShift)
CONTRIBUTING.md
LICENSE                       # Apache-2.0
```

## About Agency 42

We design, build, harden and rescue [Kubernetes](https://agency42.se/kubernetes) platforms — OpenShift, [Rancher /
RKE2](https://agency42.se/rancher), Linux underneath, [DevSecOps](https://agency42.se/devops-secops) around them. Our engineers use AI agents for
the tedious parts (logs, draft runbooks, manifest diffs) so they have more time
for judgment calls. **Agents assist; humans decide.** Nobody gets replaced by a
shell script with opinions.

Where it fits naturally we touch NIS2 / CRA / ISO 27001, but good engineering
comes first. We are **not** a Red Hat or SUSE partner — just engineers who work
on those platforms a lot.

**[agency42.se](https://agency42.se)** · **[LinkedIn](https://www.linkedin.com/company/agency42ab)**

---

<p align="center"><sub>So long, and thanks for all the pods. 🐬</sub></p>
