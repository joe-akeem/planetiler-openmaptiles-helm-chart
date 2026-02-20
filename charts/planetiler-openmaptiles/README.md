# planetiler-openmaptiles-helm-chart

Helm chart to run https://github.com/openmaptiles/planetiler-openmaptiles as a Kubernetes Job.

This chart builds OpenMapTiles with Planetiler and writes outputs to persistent storage. A new non-blocking Job is created on every `helm install` and `helm upgrade` (using the Helm release revision in the name). Helm returns immediately after creating the Job; Kubernetes runs it to completion. It supports configurable env vars/args, security contexts, resources, tolerations, node selectors, and PVCs.

## Prerequisites
- Kubernetes 1.21+ (Job `batch/v1`)
- Helm 3.8+
- PersistentVolume provisioner if you enable persistence (default: enabled)

## Quick start
Install (creates a PVC named `<release>-planetiler-openmaptiles-data` and mounts it at `/data`) and immediately runs one Job for this release operation:

Upstream recommends CLI flags. This chart defaults to `--force --download`. To build a small example area (Monaco):

```bash
helm install tiles ./ -n tiles --create-namespace \
  --set area=monaco
```

- You can target other areas similarly, e.g. `--set area=liechtenstein`.
- Prefer the `area` value to avoid shell quoting issues in zsh.
- To provide your own PBF instead of a named area, either set the appropriate upstream flag in `args` (see image docs) or use env vars like `env.DOWNLOAD_PBF_URL` if you prefer.

If you still want to manipulate the raw `args` list, here are zsh‑safe options:
- Quote the whole expression: `--set-string 'args[2]=--area=monaco'`
- Escape brackets: `--set-string args\[2\]=--area=monaco`
- Disable globbing for that command: `noglob helm install ... --set-string args[2]=--area=monaco`
- Or set the entire array at once (Helm ≥ 3.12): `--set-json args='["--force","--download","--area=monaco"]'`

### What runs when
- On `helm install`: Helm creates a normal Kubernetes Job as part of the release and returns immediately.
- On `helm upgrade`: Helm creates a new Job again (the name includes the Helm release revision like `...-r<N>`), and returns immediately.

Notes:
- Because these Jobs are not hooks, Helm does not wait for them to finish and will not fail the release if the Job later fails. Monitor the Job status/logs with `kubectl`.
- To auto-clean up finished Jobs/Pods, set `job.ttlSecondsAfterFinished`. Otherwise, historical Jobs remain until you delete them.

### Observability
List Jobs and related Pods for a release:
```bash
kubectl -n tiles get jobs -l app.kubernetes.io/instance=tiles
# Get the most recent Job name for this release
JOB=$(kubectl -n tiles get jobs -l app.kubernetes.io/instance=tiles \
  --sort-by=.status.startTime -o jsonpath='{.items[-1].metadata.name}')
kubectl -n tiles get pods -l job-name=${JOB}
```

Tail logs of the latest Pod created by the most recent Job:
```bash
kubectl -n tiles logs --tail=200 -l app.kubernetes.io/instance=tiles --selector=job-name=${JOB}
```

### Re-run the Job manually
This chart creates a new Job on each Helm action. To run it again without changing values:
- `helm upgrade tiles ./ -n tiles --reuse-values`
- Or change a no-op value (e.g., `--set dummy=$(date +%s)`) and upgrade
- Or uninstall/install the release

If you prefer ad-hoc Jobs independent of Helm, you can create your own `Job` referencing the same image/args and PVC.

### Optional cleanup/retention
- Set `job.ttlSecondsAfterFinished` to let Kubernetes auto-delete finished Jobs/Pods.
- Or leave it null to keep historical Jobs/Pods until you delete them manually.

Outputs will be written under `/data` in the container. Use a `StorageClass` that fits your cluster.

## Configuration
Common values you may want to set:

- `image.repository` (default `openmaptiles/planetiler-openmaptiles`)
- `image.tag` (defaults to the Chart's appVersion)
- `env` map to pass configuration to the image (examples below)
- `args` default CLI flags, `area`, and `extraArgs`
- `postMove.*` optional post-processing to move/rename the generated file
- `persistence.*` to configure the PVC and mount path (default mount `/data`)
- `resources` for CPU/memory
- `job.*` options: `backoffLimit`, `activeDeadlineSeconds`, `ttlSecondsAfterFinished`, `annotations`, `labels`

### Example values.yaml
```yaml
image:
  repository: ghcr.io/openmaptiles/planetiler-openmaptiles
  tag: latest
  pullPolicy: IfNotPresent

# Optional CLI args (chart defaults include --force --download)
args:
  - --force
  - --download
  - --output=/data/output.mbtiles
area: "monaco" # convenience: appends --area=monaco to args

# Environment passed to the container. Consult the upstream image for supported keys.
# Common examples:
# - DOWNLOAD_PBF_URL: the source PBF to build from
# - MIN_ZOOM / MAX_ZOOM: zoom range to generate
# - TILE_PATH, DATA_DIR, etc.
env:
  DOWNLOAD_PBF_URL: https://download.geofabrik.de/europe/germany/berlin-latest.osm.pbf

postMove:
  enabled: true
  from: "/data/output.mbtiles"
  to: "/data/tiles.mbtiles"
  runBin: "planetiler"

persistence:
  enabled: true
  storageClassName: ""
  accessModes: [ReadWriteOnce]
  size: 100Gi
  mountPath: /data

job:
  backoffLimit: 0
  activeDeadlineSeconds: null
  ttlSecondsAfterFinished: null
  annotations: {}
  labels: {}
```

### Deprecation notice about former CronJob settings
- The chart previously created a `CronJob`, then a hook‑driven one‑off `Job`. It now creates a normal non‑blocking `Job` per install/upgrade (no Helm hooks).
- Values under `cronjob.*` are ignored by the current templates. They are kept for backward compatibility for now and may be removed in a future major release.

### Persistence
- By default, the chart creates a PVC named `<release>-planetiler-openmaptiles-data` and mounts it at `/data`.
- To use an existing claim, set `persistence.existingClaim` and the chart will not create a new PVC.

### ServiceAccount
- The chart creates a dedicated ServiceAccount by default. To reuse an existing one:
  - `serviceAccount.create=false`
  - `serviceAccount.name=<your-sa>`

### Overriding command/args
If you need to override the container entrypoint or arguments:
```yaml
command: ["/bin/sh", "-lc"]
args: ["planetiler --help"]
```

### Uninstall
```bash
helm uninstall tiles -n tiles
```

If you let the chart create the PVC, it remains after uninstall by default. Delete it explicitly if desired:
```bash
kubectl -n tiles delete pvc tiles-planetiler-openmaptiles-data
```
