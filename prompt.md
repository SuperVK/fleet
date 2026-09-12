# Build Brief — Self-Hosted Nextcloud on Single-Node k3s + Flux CD (GitOps)

## 0. Role & objective

You are a senior infrastructure engineer. Produce a **complete, working GitOps repository** that  
deploys a personal Nextcloud instance on a **single-node k3s cluster on a Hetzner VPS**, reconciled  
entirely by **Flux CD** from a GitHub repository.
Optimize for: **reliability, low cost, reproducibility, security, secure secret handling**.  
Prefer **official Helm charts** over hand-rolled manifests. Where the brief conflicts with the  
pinned chart's actual `values.yaml`, **the chart wins** — read `values.yaml` for the pinned version  
before writing values.
Work in phases. After each phase, run its verification commands and stop if they fail.

* * *

## 1. Inputs / variables

Substitute these throughout. Confirm any value you cannot infer with the user before proceeding.
| Variable | Value / example |
| --- | --- |
| `DOMAIN` | `cloud.example.org` |
| `ACME_EMAIL` | `you@example.org` |
| `GITHUB_OWNER` | `<github-user-or-org>` |
| `GITHUB_REPO` | `fleet` |
| `SERVER_TYPE` | Hetzner CPX31-class (4 vCPU / 8GB RAM / 160GB NVMe) |
| `OS` | Debian 13 (trixie) |
| `NEXTCLOUD_ADMIN_USER` | `admin` |
| `STORAGE_CLASS` | `local-path` |
| `AGE_PUBLIC_KEY` | `age1...` |
| `BACKUP_TARGET` | Backblaze B2 **or** Hetzner Storage Box (S3/SFTP) |
| `RAW_ARCHIVE_TARGET` | Hetzner Object Storage (S3) **or** Hetzner Storage Box (SFTP/SMB) |

* * *

## 2. Constraints & non-goals

*   **Single node ⇒ no HA.** Use `local-path` storage class. Do **not** install Longhorn, Rook/Ceph, or any replicated storage.
    
*   **Hot data is small (~50GB).** Local NVMe only. The ~250GB RAW archive must **never** live on the node.
    
*   **Do not expose** MariaDB or Redis outside the cluster. ClusterIP only.
    
*   **Never commit plaintext secrets.** SOPS + age only; Flux decrypts in-cluster.
    
*   **Do not run** `rclone sync` during migration. Use `rclone copy` (non-destructive).
    
*   **Keep the kube API off the public internet.** Firewall + Tailscale for admin access.
    
*   **External storage** for the RAW archive is a _peek/browse_ mount, read-only. Never primary storage.
    
*   Not building: high availability, multi-tenant, external user sharing policies beyond defaults.
    

* * *

## 3. Target architecture

```
Internet ──443──► Traefik (k3s bundled, LoadBalancer)
                     │  cert-manager / Let's Encrypt (HTTP-01)
                     ▼
                  Nextcloud (Helm: nginx + php-fpm sidecar + cronjob)
                     ├── MariaDB (ClusterIP)
                     ├── Redis   (ClusterIP)
                     └── PVC (local-path, ~150Gi, ~50GB used)
Admin ──Tailscale──► k3s API (6443, never public)
Backups: k8up/restic ──► Backblaze B2 / Storage Box
RAW archive (~250GB): Hetzner Object Storage (S3) or Storage Box
                      (optionally mounted in Nextcloud as read-only external storage)
GitOps: GitHub `fleet` ──► Flux ──► cluster (continuous reconciliation)
```

* * *

## 4. Repository layout

```
fleet/
├─ .sops.yaml
├─ renovate.json
├─ README.md                      # runbook: deploy, restore, migrate
├─ clusters/prod/
│  ├─ flux-system/                # created by `flux bootstrap`
│  ├─ infrastructure.yaml         # Kustomization -> ./infrastructure
│  └─ apps.yaml                   # Kustomization -> ./apps (dependsOn infrastructure)
├─ infrastructure/
│  ├─ kustomization.yaml
│  ├─ sources/                    # HelmRepositories: nextcloud, jetstack, k8up
│  ├─ cert-manager/               # HelmRelease + ClusterIssuer
│  ├─ ingress/                    # Traefik / middleware tweaks (if needed)
│  └─ k8up/                       # k8up HelmRelease + namespace
└─ apps/nextcloud/
   ├─ kustomization.yaml
   ├─ namespace.yaml
   ├─ helmrelease.yaml
   ├─ secrets.sops.yaml           # admin pw, DB pw, SMTP, rclone.conf, restic
   ├─ backup.yaml                 # k8up Schedule + repo secret + pod annotations
   └─ import-job.yaml             # one-shot rclone OneDrive -> data dir
```

* * *

## 5. File deliverables (contents)

> Fill placeholders. Verify value keys against the **pinned chart version**.

### `.sops.yaml`

```yaml
creation_rules:
  - path_regex: .*\.sops\.ya?ml$
    encrypted_regex: "^(data|stringData)$"
    age: AGE_PUBLIC_KEY
```

### `clusters/prod/infrastructure.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  timeout: 5m
  path: ./infrastructure
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### `clusters/prod/apps.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  timeout: 10m
  path: ./apps
  prune: true
  dependsOn:
    - name: infrastructure
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### `infrastructure/sources/*.yaml` (HelmRepositories)

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: nextcloud
  namespace: flux-system
spec:
  interval: 1h
  url: https://nextcloud.github.io/helm/
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: jetstack
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.jetstack.io
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: k8up
  namespace: flux-system
spec:
  interval: 1h
  url: https://k8up-io.github.io/k8up
```

### `infrastructure/cert-manager/` (HelmRelease + ClusterIssuer)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 30m
  chart:
    spec:
      chart: cert-manager
      sourceRef:
        kind: HelmRepository
        name: jetstack
        namespace: flux-system
  values:
    installCRDs: true
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ACME_EMAIL
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

### `infrastructure/k8up/` (HelmRelease + namespace)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: k8up
  namespace: k8up
spec:
  interval: 30m
  chart:
    spec:
      chart: k8up
      sourceRef:
        kind: HelmRepository
        name: k8up
        namespace: flux-system
```

### `apps/nextcloud/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nextcloud
```

### `apps/nextcloud/helmrelease.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  interval: 30m
  chart:
    spec:
      chart: nextcloud
      version: "PIN_LATEST"      # pin; agent resolves from HelmRepository
      sourceRef:
        kind: HelmRepository
        name: nextcloud
        namespace: flux-system
  values:
    nextcloud:
      host: cloud.example.org
      existingSecret:
        enabled: true
        secretName: nextcloud-admin
        usernameKey: admin-username
        passwordKey: admin-password
      configs:
        custom.config.php: |
          <?php
          $CONFIG = array (
            'trusted_proxies'        => array('10.42.0.0/16'),
            'overwrite.cli.url'      => 'https://cloud.example.org',
            'overwrite.protocol'     => 'https',
            'overwriteprotocol'      => 'https',
            'maintenance_window_start' => 1,
          );
      phpConfigs:
        php.ini: |
          memory_limit = 512M
          upload_max_filesize = 10G
          post_max_size = 10G
      resources:
        requests: {cpu: 250m, memory: 512Mi}
        limits:   {cpu: "2",  memory: 2Gi}
    nginx:
      enabled: true
    cronjob:
      enabled: true               # REQUIRED: run the dedicated cron container
    mariadb:
      enabled: true               # verify this is the chart's bundled option
      auth:
        existingSecret: nextcloud-mariadb
    redis:
      enabled: true
    persistence:
      enabled: true
      storageClass: local-path
      size: 150Gi
    ingress:
      enabled: true
      className: traefik
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      tls: []                     # chart creates TLS via tls from host; verify
```

### `apps/nextcloud/secrets.sops.yaml` (SOPS-encrypted at rest)

*   `nextcloud-admin`: `admin-username`, `admin-password`
    
*   `nextcloud-mariadb`: `mariadb-password`, `mariadb-root-password`
    
*   `rclone-config`: `rclone.conf` (OneDrive token; used only by the import Job)
    
*   `k8up-restic`: `password`, `access-key-id`, `secret-access-key`
    

### `apps/nextcloud/backup.yaml`

```yaml
apiVersion: k8up.io/v1
kind: Schedule
metadata:
  name: nextcloud-backup
  namespace: nextcloud
spec:
  backend:
    repoPasswordSecretRef:
      name: k8up-restic
      key: password
    s3:
      endpoint: s3.eu-central-003.backblazeb2.com   # or Storage Box endpoint
      bucket: nextcloud-backups
      accessKeyIDSecretRef:
        name: k8up-restic
        key: access-key-id
      secretAccessKeySecretRef:
        name: k8up-restic
        key: secret-access-key
  backup:
    schedule: "0 2 * * *"
    keepDaily: 7
    keepWeekly: 4
  check:
    schedule: "0 8 * * 0"
  prune:
    schedule: "0 4 * * 0"
```

Also annotate the Nextcloud and MariaDB pods for **pre-backup hooks** (consistent dumps):

```
k8up.io/backup: "true"
k8up.io/backupcommand: <mysqldump | occ maintenance:mode --on ...>
```

(If annotations can't be set via the chart's values, add a documented manual/additional `Backup` CR.)

### `apps/nextcloud/import-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: onedrive-import
  namespace: nextcloud
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: rclone
          image: rclone/rclone:latest
          command: ["/bin/sh", "-c"]
          args:
            - |
              set -e
              rclone copy onedrive:Photos /data/<USER>/files/Photos \
                --include "*.{jpg,jpeg,png,heic,heif,webp}" \
                --transfers 4 --checkers 8 --tpslimit 8 \
                --retries 5 --retries-sleep 30s \
                -P --log-level INFO
          volumeMounts:
            - {name: data,        mountPath: /data}
            - {name: rclone-conf, mountPath: /config/rclone, readOnly: true}
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: nextcloud          # VERIFY actual PVC name from the chart
        - name: rclone-conf
          secret:
            secretName: rclone-config
```

### `apps/nextcloud/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: nextcloud
resources:
  - namespace.yaml
  - helmrelease.yaml
  - secrets.sops.yaml
  - backup.yaml
  - import-job.yaml
```

### `renovate.json`

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", ":dependencyDashboard", ":semanticCommits"],
  "flux": { "fileMatch": ["clusters/.+\\.ya?ml$", "apps/.+\\.ya?ml$", "infrastructure/.+\\.ya?ml$"] },
  "helm-values": { "fileMatch": ["(apps|infrastructure)/.+/values\\.ya?ml$"] },
  "packageRules": [
    { "matchDatasources": ["helm"], "groupName": "helm charts" },
    { "matchDatasources": ["docker"], "groupName": "container images" }
  ]
}
```

* * *

## 6. Implementation phases (execute in order)

### Phase 0 — Provision host

*   Create Hetzner VPS (`SERVER_TYPE`), Debian, SSH key only.
    
*   Harden:
    
    ```bash
    apt update && apt upgrade -y
    apt install -y ufw unattended-upgrades
    ufw default deny incoming; ufw default allow outgoing
    ufw allow 22/tcp; ufw allow 80/tcp; ufw allow 443/tcp
    ufw enable
    ```
    
*   Install Tailscale for admin: `curl -fsSL https://tailscale.com/install.sh | sh && tailscale up`
    
*   Allow API **only** over Tailscale: `ufw allow from 100.64.0.0/10 to any port 6443 proto tcp`
    

### Phase 1 — k3s

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 644" sh -
kubectl get nodes -o wide
kubectl get sc            # expect: local-path (default)
kubectl get svc -n kube-system   # expect: traefik (LoadBalancer)
```

### Phase 2 — Flux bootstrap

```bash
export GITHUB_TOKEN=<token>
flux bootstrap github \
  --owner=GITHUB_OWNER --repository=GITHUB_REPO \
  --branch=main --path=clusters/prod --personal
# age key for SOPS decryption:
age-keygen -o age.agekey
kubectl create secret generic sops-age -n flux-system --from-file=age.agekey
```

Commit `.sops.yaml`, `infrastructure.yaml`, `apps.yaml`, and the tree.

### Phase 3 — Infrastructure

Add `sources/`, `cert-manager/`, `k8up/`. Encrypt secrets with `sops -e -i <file>`.  
Verify: `flux get kustomizations -A` all `Ready`; `kubectl get pods -n cert-manager` Running.

### Phase 4 — Nextcloud

Add the app manifests (all encrypted secrets committed). Push; wait for reconcile.  
Config must include: `trusted_proxies`, `overwrite.cli.url`, `overwrite.protocol`, cron container  
enabled, PHP memory limit raised, sensible `maintenance_window_start`.

### Phase 5 — Backups

Deploy k8up `Schedule`, run one on-demand `Backup`, then **test a restore** into a scratch namespace.  
Do not proceed until a restore is verified.

### Phase 6 — RAW archive external storage (optional, read-only)

Configure once (not cleanly declarative in the chart). Use `occ` inside the Nextcloud pod:

```bash
occ files_external:list
# Hetzner Object Storage (S3) OR Storage Box (SFTP/SMB); read-only.
occ files_external:create "RAW Archive" <backend-id> <auth-mechanism> ...
occ files_external:option <id> readonly true
```

Document the exact commands in the README since they're not in Git.

### Phase 7 — Migration

1.  Run the `import-job` (50GB JPEGs → data dir).
    
2.  Finalize ownership + index:
    
    ```bash
    occ maintenance:mode --on
    chown -R 33:33 /var/www/html/data
    occ files:scan --all
    occ maintenance:mode --off
    ```
    
3.  RAW archive (ad hoc, not a k8s workload): `rclone copy` the RAW set to the object storage/box.
    
4.  Verify both: `rclone check <remote> <path> --one-way --size-only`.
    

### Phase 8 — Verification (see §7), then write the README runbook.

* * *

## 7. Acceptance criteria

- [ ] `flux get kustomizations -A` → all `Ready=True`.
- [ ] `kubectl get certificate -A` → `Ready=True`; `curl -I https://DOMAIN/status.php` → `200`.
- [ ] Admin Security & setup warnings: no "cron not running", no "untrusted proxy", no DB warning.
- [ ] `occ status` clean; `occ files:scan --all` returns no errors.
- [ ] `rclone size` on OneDrive source ≈ destination after copy.
- [ ] A restore from restic has been performed successfully into a scratch namespace.
- [ ] Public attack surface = 80/443 only; 6443 reachable only via Tailscale; DB/Redis not exposed.
- [ ] `git grep -i` finds **no plaintext secrets** in the repo.
- [ ] `README.md` documents: bootstrap, day-2 ops, migration, and full restore procedure.

* * *

## 8. Guardrails for the agent

*   **Never** print, echo, or commit secret values. Commit only SOPS-encrypted files.
    
*   **Never** `rclone sync` during migration — `copy` only.
    
*   **Never** make the RAW archive or object storage the Nextcloud primary data dir.
    
*   **Never** expose MariaDB/Redis via LoadBalancer/NodePort.
    
*   **Always** pin chart versions and let Renovate open PRs — no floating `latest` in production YAML.
    
*   **Always** verify manifests against the pinned chart's `values.yaml`; value keys change across  
    chart majors. If a key in this brief doesn't exist in the chart, adapt to the chart and note it.
    
*   Ask the user for any unresolved variable (DOMAIN, backup target, storage credentials) instead of inventing.
    

## 9. Definition of done

A clean checkout of the repo, applied via `flux bootstrap`, produces a working HTTPS Nextcloud with  
cron, Redis, MariaDB, verified restic backups, a reconciled GitOps loop, and a README runbook — with  
no secrets in plaintext and no RAW data on the node.