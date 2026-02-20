# planetiler-openmaptiles-helm-chart

Helm chart to run https://github.com/openmaptiles/planetiler-openmaptiles as a Kubernetes Job executed by Helm hooks.

This chart builds OpenMapTiles with Planetiler and writes outputs to persistent storage. The Job is triggered automatically on every `helm install` and `helm upgrade` via Helm hooks (`post-install,post-upgrade`). It supports configurable env vars/args, security contexts, resources, tolerations, node selectors, and PVCs.

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
- On `helm install`: a one-off Job runs after all regular resources are created (PVC, ServiceAccount, etc.).
- On `helm upgrade`: the Job runs again with the updated values.

By default the hook has `helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded`, which:
- Removes any previous hook Job before a new one is created to avoid name conflicts.
- Cleans up a successful run (the Pod and Job are removed). You can change this behavior; see Debug/Keep resources below.

### Observability
List Jobs and related Pods for a release:
```bash
kubectl -n tiles get jobs -l app.kubernetes.io/instance=tiles
kubectl -n tiles get pods -l job-name=$(kubectl -n tiles get job tiles-planetiler-openmaptiles -o jsonpath='{.metadata.name}')
```

Tail logs of the latest Pod created by the Job:
```bash
kubectl -n tiles logs --tail=200 -l app.kubernetes.io/instance=tiles --selector=job-name=tiles-planetiler-openmaptiles
```

Note: Because successful hook Jobs are cleaned up by default, fetch logs while the Job is running, or disable the cleanup as shown below.

### Re-run the Job manually
Because this is a Helm hook Job, it is triggered by Helm actions. To run it again without changing chart contents:
- `helm upgrade tiles ./ -n tiles --reuse-values` (re-applies the chart and re-triggers the hook), or
- Change a no-op value (e.g., `--set dummy=$(date +%s)`) and upgrade, or
- Uninstall/install the release.

If you prefer ad-hoc Jobs independent of Helm, you can create your own `Job` referencing the same image/args and PVC.

### Keep hook Job/Pods for debugging
To retain resources after success, override the delete policy:
```bash
helm upgrade --install tiles ./ -n tiles \
  --set-json 'job.annotations={"helm.sh/hook-delete-policy":"before-hook-creation"}'
```
You can also remove `before-hook-creation` if you want to keep historical Jobs and avoid automatic cleanup entirely. Be aware you must then manage unique names yourself; this chart’s hook Job name is static (`<release>-planetiler-openmaptiles`).

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
- The chart previously created a `CronJob`. It now creates a one-off `Job` executed via Helm hooks on install/upgrade.
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
