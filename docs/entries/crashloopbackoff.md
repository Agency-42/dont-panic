# CrashLoopBackOff: the pod starts, dies, and Kubernetes keeps trying

**Don't Panic.** `CrashLoopBackOff` is not an error. It's the kubelet telling you
"your container keeps exiting, so I'm waiting longer between restarts". The real
cause is one level down, and it nearly always leaves fingerprints.

> Longer walkthrough (planned): https://agency42.se/en/dont-panic/crashloopbackoff

## What you see

```text
NAME                      READY   STATUS             RESTARTS        AGE
my-app-7c9d8f6b5d-x2k4q   0/1     CrashLoopBackOff   6 (2m11s ago)   9m

Warning  BackOff  kubelet  Back-off restarting failed container app in pod my-app-7c9d8f6b5d-x2k4q
```

The status flips between `Running`, `Error` (or `Completed`, or `OOMKilled`) and
`CrashLoopBackOff`, and the restart count keeps climbing.

## What's going on

The container's main process exited. With `restartPolicy: Always` (every
Deployment, StatefulSet and DaemonSet) the kubelet restarts it, and each new
crash doubles the wait: 10s, 20s, 40s… capped at five minutes. If a container
then runs for ten minutes without trouble, the timer resets.

So the question is never "why CrashLoopBackOff?" but **"why did the process
exit?"** The usual suspects, roughly in order of how often they show up:

| Clue | Likely cause |
|---|---|
| Exit code `1` (or another app code), stack trace in logs | App error: bad config, missing env var, can't reach a database, failed migration |
| Reason `OOMKilled`, exit code `137` | Memory limit too low, or a leak |
| Events say `Liveness probe failed` | The probe killed a healthy-but-slow app (no startup probe, too-short timeout) |
| Exit code `0`, reason `Completed` | The process finished on purpose. That's a Job, not a Deployment |
| Exit code `126` / `127` | Command not executable / not found (wrong `command`, wrong image, missing shell in a distroless image) |
| Exit code `139` | Segfault, often a native library or architecture mismatch |
| `permission denied` / `read-only file system` in logs | `securityContext` (non-root UID, `readOnlyRootFilesystem`) vs. what the image expects |

Not the same thing, so not this entry: `ImagePullBackOff` (image never pulled),
`CreateContainerConfigError` (a referenced ConfigMap/Secret or key is missing),
`Pending` (never scheduled). Those never got as far as crashing.

## Confirm it

Replace `my-app` / `<pod>` / `app` with your namespace, pod and container.

```bash
# 1. Why did the LAST run end? Reason + exit code are the key facts.
kubectl -n my-app get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{"  "}{.lastState.terminated.reason}{"  exit="}{.lastState.terminated.exitCode}{"\n"}{end}'

# 2. Logs from the crashed run (not the current, freshly restarted one)
kubectl -n my-app logs <pod> -c app --previous

# 3. Events: probe failures, OOM, back-off
kubectl -n my-app describe pod <pod>          # read "Last State" and Events at the bottom
kubectl -n my-app get events --field-selector involvedObject.name=<pod> --sort-by=.lastTimestamp

# 4. What changed? (new image, new config, new limits)
kubectl -n my-app rollout history deploy/my-app
kubectl -n my-app get deploy my-app -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

If `--previous` is empty, the process died before writing anything. Check exit
codes `126`/`127`, the `command`/`args`, and init containers:

```bash
kubectl -n my-app logs <pod> -c <init-container-name>
```

Need a shell in something that won't stay up? Start a **copy** of the pod with
the command swapped for a shell, so you can look around and run the entrypoint
by hand:

```bash
kubectl -n my-app debug <pod> -it --copy-to=<pod>-debug --container=app -- sh
# when done:
kubectl -n my-app delete pod <pod>-debug
```

(Distroless image, no `sh`? Use the logs and exit codes, or a debug build of the image.)

## Fix

Fix the cause, not the symptom. Deleting the pod only resets the back-off timer.

- **App/config error:** fix the env var, Secret, connection string or migration,
  then roll out again. If the app dies because a dependency is unreachable, make
  it retry instead of exiting, and check whether a NetworkPolicy is in the way
  ([networkpolicy-blocks-dns](networkpolicy-blocks-dns.md)).
- **OOMKilled:** compare real usage with the limit, then raise the memory limit
  (and the request) to a measured value. If usage climbs steadily, it's a leak
  and a bigger limit only buys time.

  ```bash
  kubectl -n my-app top pod <pod> --containers
  kubectl -n my-app get pod <pod> -o jsonpath='{.spec.containers[*].resources}{"\n"}'
  ```

- **Liveness probe kills a slow starter:** add a `startupProbe` so liveness
  doesn't start counting until the app is up, and give liveness a sane
  `timeoutSeconds`/`failureThreshold`:

  ```yaml
  startupProbe:
    httpGet: { path: /healthz, port: 8080 }
    periodSeconds: 5
    failureThreshold: 30      # up to 150s to start
  livenessProbe:
    httpGet: { path: /healthz, port: 8080 }
    periodSeconds: 10
    timeoutSeconds: 3
    failureThreshold: 3
  ```

- **Exit 0 / `Completed`:** run it as a `Job` or `CronJob`, or fix the
  entrypoint so the long-running process stays in the foreground.
- **126/127:** check `command`/`args` against the image (`docker inspect` /
  `podman inspect` the entrypoint), and the image architecture vs. the node's.
- **Permission / read-only filesystem:** mount an `emptyDir` where the app writes
  (`/tmp`, cache dirs) instead of loosening the `securityContext`.

Quick escape hatch while you fix it: `kubectl -n my-app rollout undo deploy/my-app`
puts the previous ReplicaSet back, if the last rollout caused it.

## Prevent

- `startupProbe` for anything that takes more than a few seconds to boot.
  Liveness checks the process, not its dependencies (that's what readiness is for).
- Memory requests and limits based on measured usage, not guesses.
- Apps that retry dependencies with back-off instead of exiting on the first
  failed connection.
- Run staging with the same `securityContext` and NetworkPolicies as production,
  so "works in dev" means something.
- Alert on restarts, not just on down: with kube-state-metrics,
  `increase(kube_pod_container_status_restarts_total[15m]) > 3` or
  `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1`.

## Notes for OpenShift / RKE2

**OpenShift 4.x:** the classic cause is the **arbitrary UID**. The restricted SCC
runs your container as a random high UID (with group `0`), so an image that
expects to write to directories owned by a specific user crashes with
`permission denied`. Fix the image, not the SCC: make writable paths owned by
group `0` and group-writable (`chgrp -R 0 /app && chmod -R g=u /app` in the
Dockerfile), or mount an `emptyDir`. `oc debug` gives you a shell in a copy of
the workload:

```bash
oc debug deployment/my-app -n my-app
```

If the pod never gets created at all, that's SCC admission, not a crash loop:
see [openshift-unable-to-validate-against-any-scc](openshift-unable-to-validate-against-any-scc.md).

**RKE2:** the same `kubectl` steps apply. If the API is unhappy and you need the
node's view, `crictl` ships with RKE2:

```bash
export CRI_CONFIG_FILE=/var/lib/rancher/rke2/agent/etc/crictl.yaml
/var/lib/rancher/rke2/bin/crictl ps -a | grep my-app       # exited containers too
/var/lib/rancher/rke2/bin/crictl logs <container-id>
ls /var/log/pods/                                          # raw container logs per pod
```

## Related

- [The towel](../towel.md): "Pod won't start / keeps restarting" and exit codes
- [networkpolicy-blocks-dns](networkpolicy-blocks-dns.md): apps that crash because they can't resolve a dependency
- [openshift-unable-to-validate-against-any-scc](openshift-unable-to-validate-against-any-scc.md)
- [kubernetes-best-practices](https://github.com/Agency-42/kubernetes-best-practices): probes, requests/limits and securityContext defaults
- Agency 42 · [Kubernetes](https://agency42.se/kubernetes) · [Rancher / RKE2](https://agency42.se/rancher) · [agency42.se](https://agency42.se)
