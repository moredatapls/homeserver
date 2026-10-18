---
name: hostpath-volume-permissions
description: 'Diagnose and fix Kubernetes hostPath volume permission denied failures in this repository, including Loki ruler-storage, Prometheus queries.active, root-owned DirectoryOrCreate storage, and recovery after restoring a PVC. Use a restricted ownership-repair init container while preserving persistent data and non-root application users.'
---

# HostPath Volume Permissions

## Failure Mode

This repository uses statically provisioned hostPath PVs on `alderaan`, backed by `/mnt/data`. `DirectoryOrCreate` creates missing directories with the kubelet's ownership (normally root) and mode 0755. The hostPath plugin does not apply `fsGroup` ownership changes. PVC binding therefore does not guarantee that a non-root container can write its data directory.

Observed failures:

- Loki: `mkdir /var/lib/loki/rules: permission denied`, followed by `error initialising module: ruler-storage`.
- Prometheus: `open /prometheus/queries.active: permission denied`, followed by failure to initialize the active query tracker.

A manual host `chown` is an immediate repair, not a recovery-safe Kubernetes configuration.

## Diagnose

1. Read the failing container's logs and pod events. Distinguish permission errors from OOMKilled, a read-only filesystem, or an unbound claim.
2. Follow the container's `volumeMounts` to the PVC, PV, hostPath, and node affinity. Confirm that the host filesystem is mounted before starting the workload; an empty directory on the root filesystem is not recovered data.
3. Verify the application's effective UID/GID, read-only flags, directory permissions, and any `subPath`. Use live pod specifications and the pinned chart/operator version rather than assuming image defaults.
4. Inspect nearby repository implementations before editing. Preserve existing changes and keep the repair scoped to the affected workload and data volume.

## Implement

Use an idempotent init container to repair permissions before application startup. Keep the application non-root and preserve its PVC and stored data.

The existing `volume-permissions` initializers use `busybox:1.37` with:

- `runAsUser: 0`, `runAsGroup: 0`, and `runAsNonRoot: false` on the init container only, overriding the non-root pod defaults.
- `readOnlyRootFilesystem: true` and `allowPrivilegeEscalation: false`.
- All capabilities dropped, then only `CHOWN`, `DAC_OVERRIDE`, and `FOWNER` added. These permit ownership repair, traversal of restrictive restored directories, and mode changes after ownership is transferred. Do not use `privileged: true`.
- Only the affected data volume explicitly mounted, not the host root or unrelated volumes.
- Requests of 10m CPU/16Mi memory and limits of 100m CPU/32Mi memory.
- A fail-fast shell command (`sh -ec`) that does not ignore errors:

```sh
chown -Rh <uid>:<gid> <mount-path>
chmod -R u+rwX <mount-path>
chmod 0770 <mount-path>
```

Replace placeholders with verified values. Uppercase `X` preserves non-executable regular files while granting directory traversal. Ownership and mode repair must include existing data, not just the root directory, to handle restored files. Recursive repair costs startup I/O and may take time on large volumes. Use this pattern only on trusted application data; investigate unexpected symlinks or nested mounts before a recursive ownership operation.

Do not disable admission policies globally if a namespace rejects the root initializer or its capabilities. Report the restriction and agree on a permitted storage-initialization approach instead.

## Repository Implementations

### Loki

- File: `infrastructure/monitoring/loki.yaml`.
- StatefulSet: `monitoring/loki`; pod: `loki-0`.
- PVC: `monitoring-loki`; host path: `/mnt/data/monitoring/loki`.
- Volume name: `data`; mount path: `/var/lib/loki`; no subPath.
- Application and repaired ownership: UID/GID `10001:10001`.

### Prometheus

- File: `infrastructure/monitoring/prometheus.yaml`.
- HelmRelease: `monitoring/monitoring`, using kube-prometheus-stack.
- Configure `values.prometheus.prometheusSpec.initContainers`, not a direct edit to the Operator-generated StatefulSet.
- StatefulSet: `prometheus-monitoring-prometheus`; pod: `prometheus-monitoring-prometheus-0`.
- Application and repaired ownership: UID/GID `1000:2000`.
- Mount path: `/prometheus`; subPath: `prometheus-db`.
- The default Operator volume name is `prometheus-<Prometheus CR name>-db`. The chart passes init-container values through Helm `tpl`, so the existing mount name is `prometheus-{{ include "kube-prometheus-stack.prometheus.crname" . }}-db`.
- Confirm this naming and subPath against the pinned chart/operator before changes or upgrades. Do not rename the claim template to simplify mounting: that can change PVC identity and disconnect existing metrics.

Grafana and Alloy have separate storage preparation requirements. Do not claim all monitoring storage is recovery-safe based on the Loki and Prometheus fixes alone.

## Validate

1. Run focused editor/schema checks after editing. Render the pinned Helm chart when changing Helm values; editor checks alone do not validate Helm output or Operator-generated pods.
2. Check that the generated initializer mounts the same volume/subPath as the application, uses the verified UID/GID, and leaves the application non-root.
3. When cluster access is available, inspect initialization and rollout:

```sh
kubectl -n monitoring logs loki-0 -c volume-permissions
kubectl -n monitoring rollout status statefulset/loki
kubectl -n monitoring logs loki-0 -c loki
kubectl -n monitoring logs prometheus-monitoring-prometheus-0 -c volume-permissions
kubectl -n monitoring rollout status statefulset/prometheus-monitoring-prometheus
kubectl -n monitoring logs prometheus-monitoring-prometheus-0 -c prometheus
```

4. Confirm readiness, disappearance of permission errors, and continued access to existing data. If testing fresh or restored ownership, use an isolated disposable volume with restrictive root-owned files; never delete or deliberately change permissions on production data just to test recovery.
5. Update the README with symptoms, cause, automatic repair, and remaining host prerequisites. Disclose unavailable chart-rendering or live-cluster checks rather than treating static diagnostics as proof of recovery.