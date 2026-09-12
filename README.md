# fleet — Nextcloud on single-node k3s + Flux CD (GitOps)

Personal Nextcloud on a Hetzner VPS (Debian 13, CPX31-class), single-node k3s,
reconciled end-to-end by Flux CD from this repository.

- **Domain:** `cloud.victorklomp.nl` (Let's Encrypt via cert-manager, HTTP-01 through Traefik)
- **Chart pins:** nextcloud **9.2.6** (app 34.0.3), cert-manager **v1.21.2** — Renovate opens PRs for bumps
- **Storage:** `local-path` (single node, no replicated storage)
- **Secrets:** SOPS + age only, decrypted in-cluster by Flux; nothing plaintext in git
- **Backups:** handled **outside this repo** by the operator (explicitly out of scope — see §Restore for what must be covered)
- **RAW archive (~250GB):** lives in Hetzner Object Storage, mounted read-only as external storage; **never** on the node

```
Internet ──443──► Traefik (k3s bundled, LoadBalancer)
                    │  cert-manager / Let's Encrypt (HTTP-01)
                    ▼
                 Nextcloud (fpm image + nginx sidecar + cron sidecar)
                    ├── MariaDB (ClusterIP, local-path PVC 8Gi)
                    ├── Redis   (ClusterIP, cache only)
                    └── PVC nextcloud-nextcloud (local-path, 150Gi)
Admin ──Tailscale──► k3s API (6443, never public)
GitOps: github.com/SuperVK/fleet ──► Flux ──► cluster
```

## Repository layout

```
.sops.yaml                          # SOPS creation rules (age) — insert your public key
renovate.json                       # Renovate: pin + bump charts/images
clusters/prod/                      # Flux: infrastructure.yaml, apps.yaml (flux bootstrap adds flux-system/)
infrastructure/                     # HelmRepositories, cert-manager (HelmRelease + ClusterIssuer)
apps/nextcloud/                     # namespace, HelmRelease, SOPS secrets, import Job (suspended)
```

## Bootstrap (once)

Order matters: the age key and encrypted secrets must exist **before the first
push**, because `apps/nextcloud/kustomization.yaml` references
`secrets.sops.yaml` and Flux fails the `apps` Kustomization while it is missing.

### 0. Prerequisites

- Hetzner VPS (CPX31-class, Debian 13), SSH key only — done from the console.
- DNS: `A` record `cloud.victorklomp.nl` → VPS IPv4 (and `AAAA` if you want v6).
- On your workstation: `flux`, `kubectl`, `sops`, `age` (e.g. via brew/apk/apt).
- `git init -b main .` in this directory, and create the empty repo
  `SuperVK/fleet` on GitHub (no README/license — keep it empty).

### 1. Secrets (workstation)

```bash
# age keypair — private key stays on your machine (and wherever you run sops)
age-keygen -o age.agekey               # prints: public key age1...
```

Put the public key into `.sops.yaml` (replace `AGE_PUBLIC_KEY`), then:

```bash
cp apps/nextcloud/secrets.sops.yaml.example apps/nextcloud/secrets.sops.yaml
$EDITOR apps/nextcloud/secrets.sops.yaml        # replace every CHANGE_ME
# rclone.conf: run `rclone config` (onedrive backend) and paste the [onedrive] section
sops --encrypt --in-place apps/nextcloud/secrets.sops.yaml
grep -R -n "CHANGE_ME" apps/ || true            # must print nothing
```

Then push:

```bash
git add -A && git commit -m "fleet: initial manifests"
git remote add origin git@github.com:SuperVK/fleet.git
git push -u origin main
```

### 2. Harden + k3s + Tailscale (on the VPS)

```bash
apt update && apt upgrade -y
apt install -y ufw unattended-upgrades
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
# direct WireGuard for Tailscale (optional but recommended)
ufw allow 41641/udp
# kube API only over Tailscale
ufw allow from 100.64.0.0/10 to any port 6443 proto tcp
ufw enable

curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -
kubectl get nodes -o wide
kubectl get sc                  # expect local-path (default)
kubectl get svc -n kube-system  # expect traefik (LoadBalancer)
```

Notes:
- Traefik ships with k3s (a HelmChart in `kube-system`). Flux does **not** manage it; nothing needs changing.
- `--write-kubeconfig-mode 644` makes `/etc/rancher/k3s/k3s.yaml` world-readable **on the node**; 6443 is firewalled to Tailscale, so this is a usability tradeoff, not an exposure.
- Single node ⇒ flannel traffic never leaves the host; no extra firewall ports needed.

### 3. Flux bootstrap (on the VPS)

```bash
# flux CLI
curl -sSL https://fluxcd.io/install.sh | bash -s -- v2   # or download a release
export GITHUB_TOKEN=<personal access token, repo scope>

cd /opt && git clone https://github.com/SuperVK/fleet.git && cd fleet
flux bootstrap github \
  --owner=SuperVK --repository=fleet \
  --branch=main --path=clusters/prod --personal
git pull    # bootstrap committed clusters/prod/flux-system/ to the remote

# in-cluster SOPS decryption key (private key from step 1!)
kubectl create secret generic sops-age -n flux-system \
  --from-file=age.agekey=<path/to>/age.agekey
```

### 4. Watch it converge

```bash
flux get kustomizations -A          # all Ready=True
kubectl get pods -n cert-manager    # 3 Running
kubectl get certificate -A          # nextcloud-tls Ready=True
kubectl get pods -n nextcloud       # nextcloud, mariadb, redis Running
curl -I https://cloud.victorklomp.nl/status.php   # 200
```

First reconcile takes a few minutes: the `ClusterIssuer` manifest is applied
before cert-manager's CRDs exist, so Flux retries (1m interval) until the Helm
install finishes — that retry loop is expected exactly once on a fresh cluster.

Log in as `admin` with `admin-password` from your SOPS secret. Admin →
Administration settings → Basic settings should show **no** setup warnings:
cron runs via the sidecar, proxies are trusted, DB is local.

### 5. Admin access from your workstation (optional)

```bash
scp root@<vps>:/etc/rancher/k3s/k3s.yaml ~/.kube/fleet.yaml
# replace 127.0.0.1 with the VPS Tailscale IP
kubectl --kubeconfig ~/.kube/fleet.yaml get nodes
```

## Day-2 operations

```bash
# force reconcile now (interval is 10m / 30m)
flux reconcile kustomization --with-source infrastructure
flux reconcile kustomization --with-source apps

# occ (official image ships an occ wrapper that drops to www-data)
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ status
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ files:scan --all

# logs
kubectl logs -n nextcloud deploy/nextcloud -c nextcloud --tail=100
kubectl logs -n nextcloud deploy/nextcloud -c nginx    --tail=100

# suspend/resume app releases (e.g. during maintenance)
flux suspend helmrelease -n nextcloud nextcloud
flux resume  helmrelease -n nextcloud nextcloud
```

Upgrades: Renovate pins `nextcloud 9.2.6` / `cert-manager v1.21.2` /
`rclone/rclone:1.75.1` and opens PRs; merge when ready, Flux does the rest.
Nextcloud majors (34 → 35) come through the chart bump — read the chart
CHANGELOG before merging; the app runs `occ` upgrade hooks on container start.

Config changes: edit `apps/nextcloud/helmrelease.yaml` values, commit, push.
The pod template hash includes configs/phpConfigs, so pods roll on config change.

## Migration (OneDrive → Nextcloud)

1. Edit `apps/nextcloud/import-job.yaml`: set `CHANGE_ME_USER` and the
   `onedrive:` source path; commit; wait for `flux get kustomizations` to go
   Ready.
2. Run the job (it is committed suspended on purpose):

   ```bash
   kubectl patch -n nextcloud job onedrive-import -p '{"spec":{"suspend":false}}'
   kubectl logs -n nextcloud job/onedrive-import -f
   ```

   `rclone copy` (never `sync`) — non-destructive on both ends.
3. Make Nextcloud see the files:

   ```bash
   kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ maintenance:mode --on
   kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ files:scan --all
   kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ maintenance:mode --off
   ```

   The job runs as uid 33 (www-data) and writes straight into
   `<PVC>/data/<user>/files`, so no `chown` is needed.
4. Verify: `rclone size onedrive:Photos` ≈ size of `files/Photos` in the UI.

## RAW archive (Hetzner Object Storage, read-only peek)

The ~250GB RAW set never touches the node. Upload from your workstation
(ad hoc, not a k8s workload):

```bash
rclone config   # s3 backend, endpoint fsn1.your-objectstorage.com (or your region), path-style
rclone copy /path/to/raw hetzner:raw-archive --transfers 4 --checkers 8 -P
rclone check /path/to/raw hetzner:raw-archive --one-way --size-only
```

Then mount it in Nextcloud as **read-only external storage** (not declarative in
the chart — run once, it persists in the DB):

```bash
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ files_external:create \
  "RAW Archive" amazons3 amazons3::accesskey \
  --config bucket=raw-archive \
  --config hostname=fsn1.your-objectstorage.com \
  --config region=eu-central \
  --config use_path_style=true
# it prompts for the access key / secret; then lock it down:
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ files_external:option \
  <id-from-create-output> readonly true
```

`occ files_external:list` shows the id. External storage is a browse mount only —
it is never the primary data dir.

## Restore

Backups are operated outside this repo. Whatever mechanism you use must cover,
at minimum:

1. **The PVC** `nextcloud-nextcloud` (local-path: on the node it lives under
   `/var/lib/rancher/k3s/storage/...` — back that path up, or restic/k8up the
   PVC, your call),
2. **MariaDB data** (`mysqldump` into the same backup; a file-level copy of a
   running DB is not a consistent dump),
3. **This git repo** (GitHub) — the entire control plane is reproducible from it.

Full-cluster restore onto a fresh node:

```bash
# 1. Repeat Bootstrap steps 2–3 (harden, k3s, Tailscale, flux bootstrap, sops-age)
#    Flux re-creates namespaces, cert-manager, nextcloud, PVCs (empty).
# 2. Stop the app before touching data:
kubectl scale -n nextcloud deploy nextcloud --replicas=0
kubectl delete pod -n nextcloud -l app.kubernetes.io/component=cronjob 2>/dev/null || true
# 3. Restore the PVC contents into the local-path directory of the NEW PVC
#    (kubectl get pvc -n nextcloud nextcloud-nextcloud -o jsonpath='{.spec.volumeName}'
#     → find its /var/lib/rancher/k3s/storage path on the node).
# 4. Restore the DB dump:
kubectl exec -i -n nextcloud deploy/nextcloud-mariadb -- \
  sh -c 'mariadb -uroot -p"$(cat $MARIADB_ROOT_PASSWORD)" nextcloud' < dump.sql
# 5. Restart and verify:
kubectl scale -n nextcloud deploy nextcloud --replicas=1
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ status
curl -I https://cloud.victorklomp.nl/status.php
```

Secrets need no restore — they are in git, SOPS-encrypted, and Flux recreates them.

## Design notes / deviations from the original brief

Verified against the pinned charts' `values.yaml`; the chart wins where the
brief disagreed:

- **k8up/restic dropped** — backups are handled by the operator, outside the cluster repo (user decision).
- **`nextcloud.resources` does not exist** — the app-container resources key is top-level `resources`.
- **`image.flavor: fpm` is required** for `nginx.enabled: true` (default apache flavor has no php-fpm).
- **`cronjob.type: sidecar`** — chart 9.2.6 offers sidecar or CronJob; sidecar is the "dedicated cron container" running `crond` in the app pod.
- **DB credential wiring the brief missed:** with `mariadb.enabled: true` the app container reads `MYSQL_USER`/`MYSQL_PASSWORD` from the secret named by `externalDatabase.existingSecret.secretName` (chart default points at a secret nothing creates). One secret `nextcloud-mariadb` (keys `mariadb-root-password`, `mariadb-password`, `mariadb-replication-password`, `db-username`) serves both the bitnami subchart and the app env.
- **`ingress.tls: []` would mean no cert** — an explicit `tls:` entry with `secretName: nextcloud-tls` is what cert-manager's ingress shim picks up.
- **`overwrite.protocol` is not a Nextcloud config key** — used `overwriteprotocol` (+ `overwritehost`).
- **nginx body limit:** chart default `client_max_body_size 512M` overridden via `nginx.config.serverBlockCustom` to match the 10G PHP limits.
- **cert-manager `installCRDs` is deprecated** in v1.21.2 → `crds.enabled: true` (`keep: true` so an uninstall doesn't strip CRDs).
- **Import-job path correction:** the PVC is mounted with subPaths (`html/`, `data/`, …), so files go to `<PVC>/data/<user>/files`, not `<PVC>/<user>/files`. PVC name is `nextcloud-nextcloud` (fullname + `-nextcloud`).
- **PVC name for the import job** verified against chart template `nextcloud-pvc.yaml`.
- **`redis.architecture: standalone`** — the vendored bitnami redis chart defaults to `replication` (master + 3 replicas), which would triple memory use for a cache on a single node.
- **Chart renders an unused `nextcloud-db` Secret** (from default `mariadb.auth.password` = `changeme`) whenever bundled MariaDB is on. Nothing references it — the app reads `MYSQL_*` from `nextcloud-mariadb` via `externalDatabase.existingSecret` — so it is inert; just don't wire anything to it.
- **Decryption only on the `apps` Kustomization** — `infrastructure` holds no encrypted resources, so it reconciles even before `sops-age` exists.
- MariaDB and Redis are ClusterIP-only (chart default; no LoadBalancer/NodePort anywhere).
