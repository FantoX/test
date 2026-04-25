# Velero Air-Gap Backup & Restore — Deployment Guide

**System:** Air-gapped single-node Kubernetes cluster (Docker runtime)
**Stack:** Velero 1.11 · MinIO · pg_dump · Docker image backup
**Audience:** Deployment / Operations team

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Bundle Contents](#3-bundle-contents)
4. [Pre-Deployment Configuration](#4-pre-deployment-configuration)
5. [Initial Deployment (run.sh)](#5-initial-deployment-runsh)
6. [What Gets Installed](#6-what-gets-installed)
7. [Scheduled Backup Cadence](#7-scheduled-backup-cadence)
8. [How to Restore a Namespace](#8-how-to-restore-a-namespace)
9. [How to Restore PostgreSQL Data](#9-how-to-restore-postgresql-data)
10. [Day-2 Operations](#10-day-2-operations)
11. [Troubleshooting](#11-troubleshooting)
12. [Uninstall](#12-uninstall)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Air-Gapped Node                                        │
│                                                         │
│  ┌──────────────────┐   Velero backup (manifests only)  │
│  │  Kubernetes      │ ─────────────────────────────────►│
│  │  prov-eng        │                                   │
│  │  psql            │   pg_dump (DB data)               │
│  │  ingress-nginx   │ ─────────────────────────────────►│
│  │  kube-prom       │                                   │
│  │  kube-logging    │   docker save (image layers)      │
│  └──────────────────┘ ─────────────────────────────────►│
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  /backup/                                         │  │
│  │    minio-data/          ← Velero manifest store   │  │
│  │    pg-dumps/            ← PostgreSQL SQL dumps    │  │
│  │    image-backups/       ← Docker image blob store │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Three independent backup subsystems run in parallel:**

| Subsystem | What it backs up | Tool | Schedule |
|---|---|---|---|
| Velero | K8s manifests (Deployments, Services, ConfigMaps, Secrets, …) | MinIO (S3) | Every 6 h + weekly |
| pg-backup | PostgreSQL database content (pg_dump) | Filesystem | Every 6 h |
| image-backup | Docker image layers (docker save) | Filesystem | Daily at 02:00 |

> **Why three subsystems?**
> Velero cannot back up Docker image layers or hostPath volumes.
> The pg-dump handles database content that lives in a hostPath volume.
> The image-backup ensures images are available locally after a restore on an air-gapped host where `docker pull` is impossible.

---

## 2. Prerequisites

Verify these are in place **before** running `run.sh`:

| Requirement | Check command |
|---|---|
| Root / sudo access | `id` |
| Docker running | `docker info` |
| kubectl connected to cluster | `kubectl cluster-info` |
| At least 50 GB free on `/backup` partition | `df -h /backup` |
| `flock` utility | `which flock` (part of `util-linux`) |
| `curl` | `which curl` |

---

## 3. Bundle Contents

After extracting the tarball, the directory looks like:

```
velero-airgap/
├── run.sh                        ← master deployment script (start here)
├── 2-deploy-velero.sh            ← Velero + MinIO installer
├── 3-install-pg-backup.sh        ← PostgreSQL dump timer installer
├── 4-install-image-backup.sh     ← Docker image backup timer installer
├── config/
│   └── config.env                ← ALL configuration lives here
├── scripts/
│   ├── pg-backup.sh              ← runs pg_dump inside the postgres pod
│   ├── pg-restore.sh             ← restores a pg_dump into the postgres pod
│   ├── image-backup.sh           ← snapshots running Docker images to disk
│   └── image-restore.sh          ← reloads Docker images before velero restore
├── images/
│   ├── velero_velero_v1.11.1.tar
│   ├── velero_velero-plugin-for-aws_v1.7.1.tar
│   ├── minio_minio_RELEASE.*.tar
│   └── minio_mc_RELEASE.*.tar
├── binaries/
│   ├── mc
│   └── minio
└── cli/
    └── velero-v1.11.1-linux-amd64.tar.gz
```

---

## 4. Pre-Deployment Configuration

**Edit `config/config.env` before running anything. This is the only file you need to change.**

### Mandatory changes

```bash
# ── Section 1 ──────────────────────────────────────────
# MINIO_HOST_IP is auto-detected from the default-route interface.
# No change needed on single-NIC servers.
# If the node has multiple NICs or VPN, set it explicitly:
#   MINIO_HOST_IP="192.168.1.100"

# Change to a strong password (this MUST be changed)
MINIO_ROOT_PASSWORD="<STRONG_PASSWORD>"
```

> **`MINIO_HOST_IP` is now auto-detected.** At the start of `run.sh`, the detected IP is printed and you are asked to confirm before anything is installed. If the wrong IP is shown (e.g., a VPN or Docker bridge address), set `MINIO_HOST_IP` explicitly in `config/config.env` and re-run.

### Review these (change if your setup differs)

```bash
# ── Section 2 — PostgreSQL ─────────────────────────────
PG_NAMESPACE="psql"               # namespace the postgres pod runs in
PG_STATEFULSET="psql-postgresql"  # StatefulSet name
PG_POD="psql-postgresql-0"        # pod to exec pg_dump into
PG_USER="postgres"
PG_DB="hippo"                     # database name to dump
PG_PASSWORD="<db_password>"       # plain-text or leave blank to use k8s secret

# ── Section 3 — Velero schedules ──────────────────────
VELERO_CRITICAL_APP_NAMESPACES="prov-eng"   # namespaces backed up every 6h
VELERO_CLUSTER_CONFIG_NAMESPACES="prov-eng psql kube-logging ingress-nginx kube-prom"

# ── Section 8 — Image backup ───────────────────────────
IMAGE_BACKUP_NAMESPACES="prov-eng psql ingress-nginx kube-prom kube-logging"
```

> **Do not commit `config.env` to version control** — it contains passwords.

---

## 5. Initial Deployment (run.sh)

Once `config.env` is configured, run the master script as root:

```bash
cd /path/to/velero-airgap
sudo -E ./run.sh
```

The script is fully automated and runs all phases in order:

```
Phase 1 — Velero + MinIO
  preflight              checks tools, disk space, bundle layout
  load-images            docker load velero + minio tarballs
  install-cli            installs velero, mc, minio to /usr/local/bin
  install-minio          installs MinIO as a systemd service
  bootstrap-bucket       creates the velero-backups bucket, enables versioning
  install-velero         installs Velero + node-agent into the velero namespace
  patch-image-pull-policy  sets imagePullPolicy=IfNotPresent on all app workloads
  patch-postgres         annotates postgres STS to exclude hostPath volumes
  schedules              creates critical-app-6h and cluster-config-weekly schedules
  verify                 runs a sanity backup and prints status

Phase 2 — pg-backup
  installs pg-backup.sh as a systemd timer (fires every 6h)
  runs a one-shot dump to verify end-to-end connectivity

Phase 3 — image-backup
  installs image-backup.sh as a systemd timer (fires daily at 02:00)
  runs a one-shot snapshot immediately (so day-1 is covered)
```

**Expected total runtime:** 5–15 minutes depending on network and cluster size.

**If any phase fails**, fix the error and re-run `run.sh`. Every step is idempotent — safe to re-run.

---

## 6. What Gets Installed

### systemd services

| Unit | Type | What it does |
|---|---|---|
| `minio.service` | long-running | MinIO S3-compatible object store |
| `pg-backup.timer` | timer | fires `pg-backup.service` on schedule |
| `pg-backup.service` | oneshot | pg_dump → copy out → sha256 sidecar |
| `image-backup.timer` | timer | fires `image-backup.service` on schedule |
| `image-backup.service` | oneshot | docker save running pod images to blob store |

### Config files written to disk

| Path | Contains |
|---|---|
| `/etc/minio/minio.env` | MinIO credentials and listen ports |
| `/etc/pg-backup/config` | Database connection parameters |
| `/etc/image-backup/config` | Namespaces and paths for image backup |

### Binaries installed to `/usr/local/bin`

`velero`, `mc`, `minio`

### Binaries installed to `/usr/local/sbin`

`pg-backup.sh`, `pg-restore.sh`, `image-backup.sh`, `image-restore.sh`

### Backup data directories

| Directory | Contents |
|---|---|
| `/backup/minio-data/` | Velero backup objects (managed by MinIO) |
| `/backup/pg-dumps/` | `<db>_backup_<timestamp>.sql.gz` + `.sha256` sidecars |
| `/backup/image-backups/blobs/` | Content-addressed Docker image tarballs |
| `/backup/image-backups/snapshots/` | Per-run manifests mapping namespace → image → blob |

---

## 7. Scheduled Backup Cadence

```
00:00  critical-app-6h  (Velero)   ┐
00:00  pg-backup        (systemd)  │  run together every 6h
                                   │
02:00  image-backup     (systemd)  ─  daily Docker image snapshot

06:00  critical-app-6h  (Velero)   ┐
06:00  pg-backup        (systemd)  │  run together every 6h
...

Sunday 02:00  cluster-config-weekly (Velero)  weekly full-cluster snapshot
```

### Retention

| Backup type | Kept for |
|---|---|
| critical-app-6h (Velero) | 48 hours (≈ 8 restore points) |
| cluster-config-weekly (Velero) | 14 days (≈ 2 restore points) |
| pg-dumps | 7 days |
| image snapshots | 1 day (today + yesterday; configurable via `IMAGE_BACKUP_RETENTION_DAYS`) |

---

## 8. How to Restore a Namespace

Use this procedure when a namespace has been accidentally deleted or is corrupted.

### Step 1 — Pre-load Docker images

Run this **before** Velero recreates any pods. On an air-gapped host, images cannot be pulled from a registry; they must be in Docker's local cache first.

```bash
sudo image-restore.sh --namespace prov-eng
```

To see what images will be loaded:
```bash
sudo image-restore.sh --list-images --namespace prov-eng
```

To see available snapshots:
```bash
sudo image-restore.sh --list
```

### Step 2 — Run the restore

The restore command handles everything: image pre-load → velero restore → patch imagePullPolicy.

```bash
# Restore from the most recent backup (recommended)
sudo -E ./2-deploy-velero.sh restore --namespace prov-eng

# Restore from a specific backup
sudo -E ./2-deploy-velero.sh restore --namespace prov-eng --backup critical-app-6h-20240423120000
```

To see available Velero backups:
```bash
velero backup get
```

### Step 3 — Verify

```bash
kubectl -n prov-eng get pods -o wide        # all pods should be Running
kubectl -n prov-eng get events              # look for errors
velero restore describe <restore-name> --details
```

### What the restore step does internally

```
1. Lists all Completed Velero backups
2. Picks the most recent one (or uses --backup <name> if specified)
3. Calls image-restore.sh --namespace <ns>  →  docker load each required image
4. Calls velero restore create --from-backup <name> --include-namespaces <ns> --wait
5. Patches every Deployment / StatefulSet / DaemonSet in the namespace
   with imagePullPolicy=IfNotPresent  →  kubelet will never contact a registry again
```

---

## 9. How to Restore PostgreSQL Data

### List available dumps

```bash
sudo /usr/local/sbin/pg-restore.sh --list
```

### Restore the latest dump

```bash
sudo /usr/local/sbin/pg-restore.sh --latest
```

### Restore a specific dump

```bash
sudo /usr/local/sbin/pg-restore.sh /backup/pg-dumps/hippo_backup_20240423_120000.sql.gz
```

> **Safety:** The restore script takes a pre-restore safety dump first. If anything goes wrong, you can roll back to the pre-restore state using the safety dump path printed at the end.

> **Confirmation:** The script requires you to type `RESTORE` to proceed — this prevents accidental runs.

---

## 10. Day-2 Operations

### Monitoring backup health

```bash
# Velero
velero backup get                                    # list backups + status
velero schedule get                                  # confirm schedules running
velero backup-location get                           # confirm MinIO is reachable

# pg-backup
systemctl status pg-backup.timer
journalctl -u pg-backup.service -n 50
tail -50 /var/log/pg-backup.log

# image-backup
systemctl status image-backup.timer
journalctl -u image-backup.service -n 50
tail -50 /var/log/image-backup.log
image-restore.sh --list                              # see all snapshots
```

### Run a manual backup immediately

```bash
# Velero — full namespace backup now
velero backup create manual-$(date +%s) \
  --include-namespaces prov-eng \
  --ttl 48h \
  --wait

# pg-backup — dump now
sudo systemctl start pg-backup.service

# image-backup — snapshot now
sudo systemctl start image-backup.service
```

### Add a new namespace to all backups

1. Edit `config/config.env`:
   - Add the namespace to `VELERO_CRITICAL_APP_NAMESPACES` (if it needs 6h backup)
   - Add the namespace to `VELERO_CLUSTER_CONFIG_NAMESPACES` (weekly)
   - Add the namespace to `IMAGE_BACKUP_NAMESPACES`

2. Re-run the schedules and image-backup installer:
   ```bash
   sudo -E ./2-deploy-velero.sh schedules
   sudo -E ./4-install-image-backup.sh
   ```

### Patch imagePullPolicy after updating deployments

Whenever a new deployment is created or an image is updated, run:

```bash
sudo -E ./2-deploy-velero.sh patch-image-pull-policy
```

This ensures all workloads have `imagePullPolicy: IfNotPresent` so they survive a restore on an air-gapped host.

### MinIO storage management

```bash
# Check bucket usage
mc alias set local http://<MINIO_HOST_IP>:9000 minioadmin <password>
mc du local/velero-backups

# Access MinIO web console
http://<MINIO_HOST_IP>:9001
```

---

## 11. Troubleshooting

### Pods stuck in ErrImagePull after restore

The Docker image is not in the local cache. Fix:

```bash
# 1. Check which images are missing
sudo image-restore.sh --list-images --namespace prov-eng

# 2. Reload images from the latest snapshot
sudo image-restore.sh --namespace prov-eng

# 3. If blobs are missing (image-backup didn't capture it), load manually
docker load -i /path/to/image.tar

# 4. Patch pull policy to prevent future registry attempts
sudo -E ./2-deploy-velero.sh patch-image-pull-policy

# 5. Restart affected pods
kubectl -n prov-eng rollout restart deployment/<name>
```

### Velero backup stuck in "InProgress"

```bash
velero backup describe <backup-name> --details
kubectl -n velero logs deploy/velero
kubectl -n velero logs ds/node-agent
```

### MinIO not healthy

```bash
systemctl status minio
journalctl -u minio -n 80
curl -sf http://<MINIO_HOST_IP>:9000/minio/health/live
```

### pg-backup fails

```bash
journalctl -u pg-backup.service -n 100
tail -100 /var/log/pg-backup.log

# Check the postgres pod is Running
kubectl -n psql get pod psql-postgresql-0

# Test exec into the pod
kubectl -n psql exec -it psql-postgresql-0 -- pg_dump --version
```

### image-backup skips images ("not in docker cache")

This means the images are loaded in containerd but not in Docker daemon, or were GC'd.

```bash
# Verify Docker sees the images
docker image ls

# If images are missing from Docker, check containerd
crictl images

# Re-load the original image tarballs
docker load -i /path/to/original.tar

# Then re-run image-backup to snapshot them
sudo systemctl start image-backup.service
```

### Velero restore has warnings but pods are Running

Warnings (not errors) are normal for resources Velero skips (e.g., ServiceAccounts that already exist, cluster-scoped resources). Check:

```bash
velero restore describe <restore-name> --details | grep -A5 "Warnings"
```

---

## 12. Uninstall

```bash
# Remove Velero (WARNING: destroys backup CRDs and schedule history)
velero uninstall

# Stop MinIO (data in /backup/minio-data is preserved)
systemctl disable --now minio

# Remove pg-backup timer
sudo -E ./3-install-pg-backup.sh --uninstall

# Remove image-backup timer (blobs in /backup/image-backups are preserved)
sudo -E ./4-install-image-backup.sh --uninstall
```

---

## Quick Reference Card

```
DEPLOY (first time)
  sudo -E ./run.sh

MANUAL BACKUP
  velero backup create manual-$(date +%s) --include-namespaces prov-eng --ttl 48h --wait
  sudo systemctl start pg-backup.service
  sudo systemctl start image-backup.service

RESTORE NAMESPACE
  sudo -E ./2-deploy-velero.sh restore --namespace prov-eng

RESTORE POSTGRES
  sudo /usr/local/sbin/pg-restore.sh --latest

CHECK STATUS
  velero backup get
  systemctl status pg-backup.timer image-backup.timer
  image-restore.sh --list

LOGS
  journalctl -u pg-backup.service -n 50
  journalctl -u image-backup.service -n 50
  journalctl -u minio -n 50
```

---

*Document generated from velero-airgap bundle. For issues, check logs first, then re-run the relevant installer script — all steps are idempotent.*
