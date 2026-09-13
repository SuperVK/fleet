# fleet — Nextcloud + Home Assistant + Pocket ID on single-node k3s + Flux CD (GitOps)

Personal Nextcloud, Home Assistant and Pocket ID (passkey SSO/OIDC) on a Hetzner
VPS (Debian 13, CPX31-class), single-node k3s, reconciled end-to-end by Flux CD
from this repository.

- **Domains:** `cloud.victorklomp.nl` (Nextcloud), `ha.victorklomp.nl` (Home Assistant), `sso.victorklomp.nl` (Pocket ID) — Let's Encrypt via cert-manager, HTTP-01 through Traefik
- **Pins:** nextcloud **9.2.6** (app 34.0.3), home-assistant **0.3.80** (app 2026.9.1), cert-manager **v1.21.2** (charts); pocket-id image **v2.14.0**, oauth2-proxy image **v7.15.4** (both plain manifests, no official chart) — Renovate opens PRs for bumps
- **Storage:** `local-path` (single node, no replicated storage) — PVC directories live on a dedicated **Hetzner Cloud Volume** mounted at `/var/lib/rancher/k3s/storage` (see Bootstrap §2)
- **Secrets:** SOPS + age only, decrypted in-cluster by Flux; nothing plaintext in git
- **Backups:** handled **outside this repo** by the operator (explicitly out of scope — see §Restore for what must be covered)
- **RAW archive (~250GB):** lives on a **Hetzner Storage Box**, mounted read-only in Nextcloud as external storage; **never** on the node

```
Internet ──443──► Traefik (k3s bundled, LoadBalancer)
                    │  cert-manager / Let's Encrypt (HTTP-01)
                    ├─► Nextcloud (fpm image + nginx sidecar + cron sidecar)
                    │     ├── MariaDB (ClusterIP, local-path PVC 8Gi)
                    │     ├── Redis   (ClusterIP, cache only)
                    │     └── PVC nextcloud-nextcloud (local-path, 150Gi — on the Cloud Volume)
                    ├─► oauth2-proxy (SSO gate: Pocket ID OIDC) ─► Home Assistant (ClusterIP; no DB, SQLite in its PVC)
                    │     └── PVC home-assistant-home-assistant-0 (local-path, 8Gi)
                    └─► Pocket ID (OIDC provider, passkeys; SQLite in its PVC)
                          └── PVC pocket-id (local-path, 1Gi)
Admin ──Tailscale──► k3s API (6443) + SSH (22) — never public
GitOps: github.com/SuperVK/fleet ──► Flux ──► cluster
```

## Repository layout

```
.sops.yaml                          # SOPS creation rules (age) — insert your public key
renovate.json                       # Renovate: pin + bump charts/images
clusters/prod/                      # Flux: infrastructure.yaml, issuers.yaml, apps.yaml (flux bootstrap adds flux-system/)
infrastructure/                     # HelmRepositories, cert-manager (HelmRelease), issuers/ (ClusterIssuer)
apps/nextcloud/                     # namespace, HelmRelease, SOPS secrets, import Job (suspended)
apps/home-assistant/                # namespace, HelmRelease (chart ingress off; own Ingress routes via oauth2-proxy), oauth2-proxy Deployment/Service, SOPS secret (Pocket ID client credentials)
apps/pocket-id/                     # namespace, plain Deployment/Service/PVC/Ingress (no official chart), SOPS secret (ENCRYPTION_KEY)
```

## Bootstrap (once)

Order matters: the age key and encrypted secrets must exist **before the first
push**, because `apps/nextcloud/kustomization.yaml` (and `apps/pocket-id/`)
reference `secrets.sops.yaml` and Flux fails the `apps` Kustomization while it
is missing.

### 0. Prerequisites

- Hetzner VPS (CAX11-class, Debian 13), SSH key only — done from the console.
- DNS: `A` records `cloud.victorklomp.nl`, `ha.victorklomp.nl` and `sso.victorklomp.nl` → VPS IPv4 (and `AAAA` if you want v6).
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

### 2. Harden + data volume + k3s + Tailscale (on the VPS)

```bash
apt update && apt upgrade -y
apt install -y ufw unattended-upgrades
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp    # TEMPORARY: needed for bootstrap over the public IP; closed to Tailscale-only below
ufw allow 80/tcp
ufw allow 443/tcp
# direct WireGuard for Tailscale (optional but recommended)
ufw allow 41641/udp
# kube API only over Tailscale (SSH gets the same treatment below,
# once Tailscale is up)
ufw allow from 100.64.0.0/10 to any port 6443 proto tcp
ufw enable

curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip -4    # note the node's Tailscale IP (also visible in the admin console)

# Data volume for every PVC: k3s's bundled local-path-provisioner places all
# PVC directories under /var/lib/rancher/k3s/storage, so mounting the Hetzner
# Cloud Volume AT that path puts every PVC on it — zero manifest changes.
# Must be mounted before k3s provisions the first PVC.
ls -l /dev/disk/by-id/ | grep HC_Volume    # e.g. scsi-0HC_Volume_106859350
mkfs.ext4 /dev/disk/by-id/scsi-0HC_Volume_106859350   # SKIP if the volume is already ext4 — it survives VPS reinstalls
mkdir -p /var/lib/rancher/k3s/storage
echo '/dev/disk/by-id/scsi-0HC_Volume_106859350 /var/lib/rancher/k3s/storage ext4 discard,nofail,defaults 0 0' >> /etc/fstab
mount /var/lib/rancher/k3s/storage
df -h /var/lib/rancher/k3s/storage         # must show the volume's size, not the root disk

# SSH: Tailscale-only, like 6443. SAFETY: first open a second session over
# Tailscale from your workstation (ssh root@<tailscale-ip>) and confirm it
# works; only then run these — from that session. Your current public-IP
# session survives (ufw permits established connections) but could not
# reconnect if it dropped.
ufw delete allow 22/tcp
ufw allow from 100.64.0.0/10 to any port 22 proto tcp

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -
kubectl get nodes -o wide
kubectl get sc                  # expect local-path (default)
kubectl get svc -n kube-system  # expect traefik (LoadBalancer)
```

Notes:
- Traefik ships with k3s (a HelmChart in `kube-system`). Flux does **not** manage it; nothing needs changing.
- `--write-kubeconfig-mode 644` makes `/etc/rancher/k3s/k3s.yaml` world-readable **on the node**; 6443 is firewalled to Tailscale, so this is a usability tradeoff, not an exposure.
- Single node ⇒ flannel traffic never leaves the host; no extra firewall ports needed.
- **Cloud-init auto-mount:** if the volume was attached at install time, cloud-init may have mounted it at `/mnt/HC_Volume_<id>` (check `grep HC_Volume /etc/fstab`). Keep only ONE fstab entry: `umount /mnt/HC_Volume_<id>`, delete that line, use the `/var/lib/rancher/k3s/storage` one instead.
- **Volume size vs PVC size:** local-path does not enforce the requested PVC capacity against the actual disk — the 150Gi Nextcloud PVC binds fine on a smaller volume. If data outgrows the volume, resize it in Hetzner Console and `resize2fs` the device (both online, no rebuild).
- **Break-glass:** with SSH Tailscale-only there is no way in if Tailscale breaks — that is the deliberate tradeoff. Recovery path is Hetzner Console → server → VNC console (or mounting the disk in rescue mode).

### 3. Flux bootstrap (on the VPS)

```bash
# flux CLI
curl -sSL https://fluxcd.io/install.sh | bash  # or download a release
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
kubectl get certificate -A          # nextcloud-tls, home-assistant-tls, pocket-id-tls Ready=True
kubectl get pods -n nextcloud       # nextcloud, mariadb, redis Running
kubectl get pods -n home-assistant  # home-assistant-0, oauth2-proxy Running
kubectl get pods -n pocket-id       # pocket-id Running
kubectl get pvc -A                  # 4 Bound (nextcloud, mariadb, redis, HA) + pocket-id
curl -I https://cloud.victorklomp.nl/status.php   # 200
curl -I https://ha.victorklomp.nl                 # 302 to the Pocket ID login (see §Day-2: register the OIDC client first)
curl -I https://sso.victorklomp.nl                # 200 (or 302 to /setup)
```

First reconcile takes a few minutes and happens in strict order:
`infrastructure` (installs cert-manager + its CRDs) → `issuers` (the
ClusterIssuer, which dry-run-fails until those CRDs exist — that's why it is a
separate Kustomization with `dependsOn`) → `apps` (Nextcloud, Home Assistant, Pocket ID).
Co-locating the ClusterIssuer with the HelmRelease deadlocks: one dry-run
failure aborts the whole Kustomization apply, so the HelmRelease that provides
the CRDs never lands.

Log in to Nextcloud as `admin` with `admin-password` from your SOPS secret.
Admin → Administration settings → Basic settings should show **no** setup
warnings: cron runs via the sidecar, proxies are trusted, DB is local.

Home Assistant: open https://ha.victorklomp.nl — behind the oauth2-proxy SSO
gate, so authenticate with your Pocket ID passkey first; the onboarding flow
then creates the first (owner) user and stores it in its PVC. The gate needs
its OIDC client registered before the first login works — see §Day-2.

Pocket ID: open https://sso.victorklomp.nl/setup — the first visit registers the
admin passkey and claims the instance (same pattern as HA: no admin secret in
git; the only secret is the SOPS `ENCRYPTION_KEY`). OIDC clients for other apps
are then added in its admin UI.


### 5. Admin access from your workstation (optional)

```bash
# SSH is Tailscale-only, so scp goes over the tailnet as well
scp root@<tailscale-ip-or-name>:/etc/rancher/k3s/k3s.yaml ~/.kube/fleet.yaml
# replace 127.0.0.1 in the copied file with the node's Tailscale IP
# (tailscale ip -4 on the node)
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
flux suspend helmrelease -n home-assistant home-assistant
flux resume  helmrelease -n home-assistant home-assistant
```

Home Assistant day-2: config lives in its PVC (`/config`), not in git — edit
via the UI or `kubectl exec -it -n home-assistant pod/home-assistant-0 -- bash`
and change files directly (then delete the pod to restart). StatefulSet, so
the pod is `home-assistant-0` and chart upgrades recreate it in place. Logs:
`kubectl logs -n home-assistant home-assistant-0 -f` (init container:
`-c setup-config`; HA's own log file is `/config/home-assistant.log`).

**Home Assistant SSO gate (oauth2-proxy + Pocket ID).** `ha.victorklomp.nl`
is routed Traefik → oauth2-proxy → HA; the web UI requires a Pocket ID
passkey session before HA's own login page is even served. `^/api/` bypasses
the gate (`OAUTH2_PROXY_SKIP_AUTH_ROUTES` in `apps/home-assistant/oauth2-proxy.yaml`)
so the companion apps (HA's own token auth) and integration webhooks keep
working — HA still enforces authentication on `/api/` itself.

One-time setup (the committed SOPS secret ships placeholders):

```bash
# 1. Pocket ID admin UI (sso.victorklomp.nl) → API Keys → new OIDC client,
#    callback URL: https://ha.victorklomp.nl/oauth2/callback
# 2. Fill in the generated credentials:
SOPS_AGE_KEY_FILE=age.agekey sops edit apps/home-assistant/secrets.sops.yaml
#    client-id / client-secret ← Pocket ID; leave cookie-secret as is
# 3. Commit, push; Flux rolls the Deployment within 10m.
```

Rotating `cookie-secret` only drops every active SSO session (users
re-authenticate). Gate logs: `kubectl logs -n home-assistant deploy/oauth2-proxy`.

**Restoring a backup wipes the reverse-proxy trust.** An imported backup
replaces `/config/.storage/http` with the source instance's settings, so
`trusted_proxies` no longer covers the pod CIDR and every request through
Traefik gets `400` (log: `Received X-Forwarded-For header from an untrusted
proxy`). Fix after every restore:

```bash
kubectl --kubeconfig ~/.kube/fleet.yaml --insecure-skip-tls-verify exec -n home-assistant home-assistant-0 -- \
  python3 -c 'import json; p="/config/.storage/http"; d=json.load(open(p)); [d["data"][s].update(trusted_proxies=["10.42.0.0/16"]) for s in ("stable","pending") if d["data"].get(s)]; json.dump(d,open(p,"w"),indent=2)'
kubectl --kubeconfig ~/.kube/fleet.yaml --insecure-skip-tls-verify delete pod -n home-assistant home-assistant-0
```

Upgrades: Renovate pins `nextcloud 9.2.6` / `home-assistant 0.3.80` /
`cert-manager v1.21.2` / `rclone/rclone:1.75.1` / `oauth2-proxy v7.15.4` and
opens PRs; merge when ready, Flux does the rest. Nextcloud majors (34 → 35)
come through the chart bump — read the chart CHANGELOG before merging; the app
runs `occ` upgrade hooks on container start. Home Assistant chart majors are
rare (0.x) and each bump carries a new HA release (monthly); the app migrates
its DB on start.

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

## RAW archive (Hetzner Storage Box, read-only peek)

The ~250GB RAW set never touches the node. It lives on a Hetzner **Storage Box**
(cheaper than Object Storage at this size). Upload from your workstation
(ad hoc, not a k8s workload) over SFTP — port 22, always on, nothing to enable:

```bash
rclone config   # sftp backend: host uXXXXX.your-storagebox.de, port 22, user uXXXXX, box password
rclone copy /path/to/raw storagebox:raw-archive --transfers 4 --checkers 8 -P
rclone check /path/to/raw storagebox:raw-archive --one-way --size-only
```

Then mount it in Nextcloud as **read-only external storage** over WebDAV
(not declarative in the chart — run once, it persists in the DB). WebDAV, not
SMB: the official Nextcloud image ships no `php-smbclient`, so the SMB backend
would need a custom image — the `dav` backend is pure PHP.

Prerequisite: enable WebDAV for the box in Hetzner Console (Storage Box →
Settings); activation takes a few minutes. Do **not** tick the box's own
read-only option — a read-only box serves plain HTTP GETs and WebDAV clients
(Nextcloud's included) cannot even list it. The read-only guarantee comes from
the `readonly` mount option applied below.

1. Create the mount — backend id `dav`, auth `password::password`; all options
   (storage **and** auth) go via `--config`, so nothing is prompted:

```bash
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ files_external:create \
  "RAW Archive" dav password::password \
  --config host=uXXXXX.your-storagebox.de \
  --config root=/raw-archive \
  --config secure=true \
  --config user=uXXXXX \
  --config password='<storage box password>'
# then lock it down (id comes from the create output):
kubectl exec -n nextcloud deploy/nextcloud -c nextcloud -- occ files_external:option \
  <id-from-create-output> readonly true
```

`occ files_external:list` shows the id (and lets you verify the config). External
storage is a browse mount only — it is never the primary data dir.

## Restore

Backups are operated outside this repo. Whatever mechanism you use must cover,
at minimum:

1. **The Nextcloud PVC** `nextcloud-nextcloud` (local-path: on the node it lives
   under `/var/lib/rancher/k3s/storage/...` — that is the **Cloud Volume**
   mount, so back the volume up, or restic/k8up the PVC, your call),
2. **The Home Assistant PVC** `home-assistant-home-assistant-0` — its SQLite DB
   (`/config/home-assistant_v2.db`) and all YAML config live there; a
   file-level copy taken while the pod is stopped is consistent (single
   SQLite writer),
3. **MariaDB data** (`mysqldump` into the same backup; a file-level copy of a
   running DB is not a consistent dump),
4. **The Pocket ID PVC** `pocket-id` — SQLite DB with users, passkeys and OIDC
   clients, all encrypted at rest with the `ENCRYPTION_KEY` from the SOPS secret
   (which is in git). Losing that key invalidates every client credential, so
   the PVC and the repo secret must be restored as a pair,
5. **This git repo** (GitHub) — the entire control plane is reproducible from it.

Full-cluster restore onto a fresh node:

```bash
# 1. Repeat Bootstrap steps 2–3 (harden, data volume, k3s, Tailscale, flux bootstrap, sops-age)
#    Flux re-creates namespaces, cert-manager, nextcloud, home-assistant, pocket-id, PVCs (empty).
# 2. Stop the app before touching data:
kubectl scale -n nextcloud deploy nextcloud --replicas=0
kubectl delete pod -n nextcloud -l app.kubernetes.io/component=cronjob 2>/dev/null || true
# 3. Restore the PVC contents into the local-path directory of the NEW PVC
#    (kubectl get pvc -n nextcloud nextcloud-nextcloud -o jsonpath='{.spec.volumeName}'
#     → find its /var/lib/rancher/k3s/storage path — on the Cloud Volume — on the node).
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
- **`nextcloud.phpConfigs` is inert when `nginx.enabled: true`** — the chart writes it to `/usr/local/etc/php-fpm.d/php.ini`, but `php-fpm.conf` includes `php-fpm.d/*.conf` only, so fpm never reads it (the old `memory_limit`/`upload_max_filesize` block silently never applied). PHP ini overrides instead live in the `nextcloud-php-ini` ConfigMap (`php-conf.yaml`), mounted into `/usr/local/etc/php/conf.d/zz-nextcloud.ini` via `nextcloud.extraVolumes`/`extraVolumeMounts` — conf.d is scanned alphabetically, so the `zz-` prefix loads last and overrides the image's `opcache-recommended.ini`. The chart mounts it into the app, nginx and cron containers alike.
- **opcache JIT disabled** (`opcache.jit=disable`) — image `34.0.3-fpm` (PHP 8.5.10) ships `opcache.jit=1255` (tracing). 2026-09-13: right after installing the `user_oidc` app, every fpm worker began SIGSEGVing on every request (`/status.php` probes 502'd → nginx sidecar crashloop; CLI `occ` unaffected since `opcache.enable_cli=Off`). Disabling the app didn't help — the poisoned opcache/JIT shared memory persists until fpm restarts; only a pod restart recovered. Tracing JIT has a recurring segfault history on fresh PHP builds and Nextcloud is I/O-bound, so it stays off.
- **cert-manager `installCRDs` is deprecated** in v1.21.2 → `crds.enabled: true` (`keep: true` so an uninstall doesn't strip CRDs).
- **Import-job path correction:** the PVC is mounted with subPaths (`html/`, `data/`, …), so files go to `<PVC>/data/<user>/files`, not `<PVC>/<user>/files`. PVC name is `nextcloud-nextcloud` (fullname + `-nextcloud`).
- **PVC name for the import job** verified against chart template `nextcloud-pvc.yaml`.
- **`redis.architecture: standalone`** — the vendored bitnami redis chart defaults to `replication` (master + 3 replicas), which would triple memory use for a cache on a single node.
- **Chart renders an unused `nextcloud-db` Secret** (from default `mariadb.auth.password` = `changeme`) whenever bundled MariaDB is on. Nothing references it — the app reads `MYSQL_*` from `nextcloud-mariadb` via `externalDatabase.existingSecret` — so it is inert; just don't wire anything to it.
- **Decryption only on the `apps` Kustomization** — `infrastructure` holds no encrypted resources, so it reconciles even before `sops-age` exists.
- MariaDB and Redis are ClusterIP-only (chart default; no LoadBalancer/NodePort anywhere).

Pocket ID (image v2.14.0, plain Kustomize manifests — no chart):

- **No Helm chart on purpose** — every published chart is community-maintained (matslarson, anza-labs, TrueCharts, …) and unofficial; the app is a single Go binary with SQLite, so Deployment + Service + PVC + Ingress is the whole story. Renovate still bumps the image (`docker.fileMatch` covers `apps/**/*.yaml`); the image is multi-arch and the node is arm64.
- **Deployment, not StatefulSet** — one replica on an RWO local-path PVC; on a single node a StatefulSet buys nothing (Pocket ID's own docker-compose is one container).
- **`TRUST_PROXY=10.42.0.0/16`** — same pod-CIDR rationale as the Nextcloud `trusted_proxies`; without it, rate limiting and the audit log only ever see Traefik's pod IP.
- **`ENCRYPTION_KEY` in SOPS** — Pocket ID encrypts its token-signing keys with it; the ciphertext lives in the PVC and the key in git, so §Restore treats them as a pair. Unlike nextcloud there is nothing user-specific to fill in, so the encrypted secret is generated and committed directly (template kept as `secrets.sops.yaml.example`).
- **Healthcheck via exec** (`/app/pocket-id healthcheck`, the command from Pocket ID's own compose file) — the app exposes no HTTP health endpoint to probe instead.
- **HTTPS mandatory** — WebAuthn requires a secure context; traefik + cert-manager already provide it, so the ingress is plain.

Home Assistant (chart 0.3.80, values verified against its `values.yaml`):

- **Chart choice:** no official HA chart exists; k8s-at-home is archived. The pajikos chart is auto-published with each HA release (2026.9.1 here), pins nothing weird, and renders a plain StatefulSet + Service + Ingress — verified by templating 0.3.80 locally.
- **`hostNetwork: false` on purpose** — hostNetwork only buys mDNS/SSDP discovery on a home LAN; this HA runs on a datacenter VPS with no LAN, so it's pure downside (port conflicts with the host, non-cluster DNS). Integrations here are outbound-only (cloud APIs, MQTT, webhook over the ingress).
- **`configuration.enabled: true`** makes the chart seed `configuration.yaml` and, on a fresh install only, `/config/.storage/http` with `use_x_forwarded_for` + `trusted_proxies: 10.42.0.0/16` (the pod CIDR Traefik hops arrive from — same rationale as the Nextcloud `trusted_proxies`). HA 2026.8+ moved these http settings from `configuration.yaml` into `.storage`; the chart writes the storage file only on first boot, so later UI edits win.
- **PVC name** is `home-assistant-home-assistant-0`: the chart's default controller is a StatefulSet, so `persistence.*` renders as a `volumeClaimTemplates` entry (claim `home-assistant` + pod `home-assistant-0`), not a standalone PVC.
- **No SOPS secret needed** — the chart has no admin-credential injection; HA's onboarding creates the owner user on first visit and keeps it in the PVC.
- **SQLite kept** (chart default) — the recorder on a single-instance HA with a few hundred entities is well within SQLite's envelope; a separate DB would be another stateful pod for no gain.

Infra-level decisions (not from any chart's values):

- **PVCs on a dedicated Hetzner Cloud Volume**, mounted at `/var/lib/rancher/k3s/storage` — the exact path k3s's local-path-provisioner provisions into. Zero manifest changes (the `local-path` storage class is untouched), and the volume survives VPS reinstalls while its mount step is part of Bootstrap §2.
- **RAW archive on a Storage Box, not Object Storage** (user decision — cheaper at ~250GB). Upload via SFTP (rclone), browse via the files_external `dav` (WebDAV) backend: the official Nextcloud image has no `php-smbclient`, so SMB would require a custom image. Backend/auth ids (`dav`, `password::password`) and the `--config`-carries-auth-options behavior verified against `apps/files_external/lib/Command/Create.php` in server 34.
