# Discourse — EKS Deployment

This repo holds the ArgoCD/ACK manifests and container build that run **[Discourse](https://www.discourse.org/)** self-hosted on the **`mzla-eks-workloads01`** EKS cluster (`eu-central-1`, account `668807881758`). The forum is exposed at **<https://discourse.thunderbird.net>** via Cloudflare Tunnel.

It is the planned replacement for Topicbox; data migration is a separate follow-up phase.

> **Audience:** SREs landing here for the first time should read *Architecture* + *Where things live* to understand the system. Senior SREs should jump to *Gotchas* and *Operational runbooks* for the things that bit us during initial deployment.

**Source of truth for the deployment plan and decision history:** TBD (link the platform-infrastructure tracking issue once filed).

---

## 1. What is Discourse?

Discourse is a Rails (Ruby 3.4) forum platform. It is opinionated about how it's run — upstream supports Docker via the [`discourse_docker`](https://github.com/discourse/discourse_docker) launcher and [explicitly does not officially support Kubernetes](https://meta.discourse.org/t/installing-on-kubernetes/49329). The container built from `container/` here is a `discourse_docker`-bootstrapped image with our plugin set baked in; we then run it on K8s with the data tier (Postgres, Redis, blob storage) externalised to AWS managed services.

### Components we run

| Component | Image | Purpose |
|-----------|-------|---------|
| `discourse-web` | `668807881758.dkr.ecr.eu-central-1.amazonaws.com/discourse:<tag>` | Puma + Rails app on `:3000` (HTTP), `:7001` (Prometheus). 2 replicas. |
| `discourse-sidekiq` | same image, `bundle exec sidekiq` | Background-job consumer (mail, search indexing, digest builds, etc). 1 replica. |
| `discourse-tunnel` | `cloudflare/cloudflared` (managed by `cloudflare-operator`) | Outbound tunnel. Zero inbound ports open on the cluster. |
| `discourse-postgres` (RDS) | `postgres:16.6` (AWS RDS) | Application database. |
| `discourse-redis` (ElastiCache) | `redis:7.1` (AWS ElastiCache) | Cache + Sidekiq queue. AUTH-token enabled, transit encryption on. |
| `mzla-discourse-uploads` (S3) | — | Object storage for avatars, attachments, post images, backups. Pod access via IRSA. |
| `email-smtp.eu-central-1.amazonaws.com` (SES) | — | Outbound SMTP. Currently in SES sandbox; production access required pre-launch. |

---

## 2. Architecture

```
            Internet
                │
                ▼
  ┌───────────────────────────────┐
  │  Cloudflare edge (CDN + bot   │
  │  protection on thunderbird.net│
  │  zone)                        │
  └───────────────────────────────┘
                │  outbound tunnel
                ▼
  ┌───────────────────────────────┐    namespace: discourse
  │  discourse-tunnel             │    on mzla-eks-workloads01
  │  (cloudflared)                │
  └───────────────────────────────┘
                │  http://discourse-web:80 → :3000
                ▼
  ┌───────────────────────────────┐
  │  discourse-web (Puma + Rails) │──── :7001/metrics ──► VMAgent
  │  - HTTP API + UI              │
  └───────────────────────────────┘
            │            │
            │            └──► s3:mzla-discourse-uploads (IRSA)
            │
   ┌────────┴────────┐
   ▼                 ▼
┌──────────────┐    ┌──────────────────┐
│ ElastiCache  │    │  RDS Postgres    │
│ Redis 7.1    │    │  discourse-      │
│ AUTH + TLS   │    │  postgres        │
│              │    │  16.6, db.t4g.   │
└──────────────┘    │  medium, 30Gi    │
        ▲           │  gp3, encrypted  │
        │           └──────────────────┘
        │ jobs                ▲
        │                     │
   ┌────┴────────────────┐    │
   │ discourse-sidekiq   │────┘
   │ (bundle exec sidekiq│
   └─────────────────────┘
```

Every external request comes in through Cloudflare → `cloudflared` → `discourse-web` Service → web pod. There is no inbound LB or Ingress on the cluster.

---

## 3. Where things live

### In this repo

```
discourse-deploy/
├── README.md                                       you are here
├── container/
│   ├── Dockerfile                                  bootstrap wrapper invoking discourse_docker launcher
│   ├── containers/app.yml                          discourse_docker config (plugins, hooks)
│   └── .github/workflows/build.yml                 build + push to ECR on tag
├── argocd/
│   ├── aws-resources/
│   │   ├── rds-postgres.yaml                       ACK DBSubnetGroup + DBInstance (postgres 16)
│   │   ├── elasticache-redis.yaml                  ACK CacheSubnetGroup + ReplicationGroup (redis 7.1)
│   │   └── s3-bucket.yaml                          ACK Bucket: mzla-discourse-uploads
│   ├── secrets/
│   │   ├── discourse-db-credentials.yaml           ES → mzla/discourse/db
│   │   ├── discourse-redis-credentials.yaml        ES → mzla/discourse/redis
│   │   ├── discourse-app-secrets.yaml              ES → mzla/discourse/app (SECRET_KEY_BASE, admin)
│   │   ├── discourse-smtp-credentials.yaml         ES → mzla/discourse/smtp (SES)
│   │   └── cloudflare-credentials.yaml             ES → mzla/twenty/cloudflare (shared)
│   └── workloads/
│       ├── default-serviceaccount.yaml             IRSA annotation overlay on ns default SA
│       ├── rds-bootstrap-job.yaml                  CREATE USER + GRANTs + CREATE EXTENSION
│       ├── discourse-config.yaml                   ConfigMap of DISCOURSE_* runtime env
│       ├── discourse-web.yaml                      Deployment + Service for Puma
│       ├── discourse-sidekiq.yaml                  Deployment for Sidekiq workers
│       ├── cloudflare-tunnel.yaml                  Tunnel + TunnelBinding (discourse.thunderbird.net)
│       └── vmpodscrape.yaml                        VictoriaMetrics scrape for :7001
```

### In `thunderbird/platform-infrastructure`

- `argocd/projects/discourse.yaml` — ArgoCD `AppProject` scoping us to the workloads cluster's `discourse` namespace.
- `argocd/workloads/apps/discourse-app-of-apps.yaml` — sync pointer to *this repo's* `argocd/` directory.
- `pulumi/environments/mzla-workloads/config.prod.yaml` — defines the `workloads-prod-discourse-default` IRSA role (S3 access for uploads), the `workloads-prod-discourse-ses` IAM user (SES SMTP), and the four AWS Secrets Manager secrets at `mzla/discourse/{db,redis,app,smtp}`.

### In AWS

| Service | Resource | How |
|---|---|---|
| RDS | `discourse-postgres` (postgres 16.6) | ACK |
| ElastiCache | `discourse-redis` (redis 7.1) | ACK |
| S3 | `mzla-discourse-uploads` | ACK |
| IAM | role `workloads-prod-discourse-default` (IRSA) | Pulumi |
| IAM | user `workloads-prod-discourse-ses` + access key | Pulumi |
| Secrets Manager | `mzla/discourse/db`, `/redis`, `/app`, `/smtp` | Pulumi |
| Secrets Manager | `mzla/twenty/cloudflare` (shared CF token) | Pulumi (twenty's stack) |
| SES | sender identity `noreply@thunderbird.net` from verified domain `thunderbird.net` | manual / Pulumi (TBD) |

---

## 4. Bring-up runbook

Order matters because the ACK CRDs report endpoints in their `status` only *after* AWS reconciles them; Discourse needs those endpoints in `discourse-config` and `rds-bootstrap-job` to come up.

1. **Pulumi up** in `mzla-workloads` to create IRSA role, SES IAM user, and the four SM secrets (passwords are `RandomPassword`-generated).
2. **Register the app** by merging `argocd/projects/discourse.yaml` and `argocd/workloads/apps/discourse-app-of-apps.yaml` in `platform-infrastructure`. ArgoCD picks up this repo and starts syncing; sync waves 0–2 land (ServiceAccount, ExternalSecrets, ACK subnet groups + S3) immediately. Wave 3 (DBInstance, ReplicationGroup) takes 10–15 minutes.
3. **Patch endpoints** once RDS + ElastiCache are `available`:
   ```
   kubectl get dbinstance discourse-postgres -n discourse -o jsonpath='{.status.endpoint.address}'
   kubectl get replicationgroup discourse-redis -n discourse -o jsonpath='{.status.nodeGroups[0].primaryEndpoint.address}'
   ```
   Update `discourse-config.yaml` `DISCOURSE_DB_HOST` / `DISCOURSE_REDIS_HOST` and `rds-bootstrap-job.yaml` `PGHOST` with the resolved values, commit, push. ArgoCD re-syncs.
4. **Wave 4** (`rds-bootstrap-job`) creates `discourse_app_user`, grants, and the `hstore` + `pg_trgm` extensions.
5. **Wave 5** brings up `discourse-web`, `discourse-sidekiq`, the Cloudflare tunnel, and the VMPodScrape. First boot of `discourse-web` runs migrations (~3–5 min); watch `kubectl logs -f deploy/discourse-web -n discourse`.
6. **Smoke** at <https://discourse.thunderbird.net> — log in as the developer email with the `adminPassword` from `mzla/discourse/app`, create a category, upload an image (verifies S3 + IRSA), trigger a test email from `/admin/email`.
7. **SES production access** — currently in sandbox. Re-file the production-access request via the SES console (the prior case `177704803400657` was DENIED) before allowing user signups.

---

## 5. Gotchas

- **SES sandbox**: in sandbox, Discourse can only send to verified addresses. Add staff emails to the SES verified-identity list during build/test, then move out of sandbox before launch.
- **No dedicated SGs**: RDS and ElastiCache attach to the EKS cluster SG (`sg-0ea4b5695515a4b50`). EKS's intra-SG self-reference rule is what makes pod-to-managed-service connectivity work; don't change this without also adding explicit SG rules.
- **Patching endpoints is two-phase by design**: ACK doesn't expose the RDS / ElastiCache endpoints until *after* creation. We commit `<TBD>` placeholders, sync, wait, patch, sync. If you try to lay everything down in a single PR with placeholders, web pods will CrashLoop until the patch lands.
- **Cloudflare token is shared with twenty**: `mzla/twenty/cloudflare` covers both apps (same Cloudflare account). Don't create a `mzla/discourse/cloudflare` — it'd just duplicate.
- **Sidekiq image == web image**: same OCI image, different `command:`. Don't fork or re-bootstrap; let the Dockerfile produce one artifact and override the entrypoint at the pod level.
- **Postgres extensions**: Discourse needs `hstore` and `pg_trgm`. The bootstrap Job creates these as the master user — the app user can't `CREATE EXTENSION` on RDS.
- **Topicbox import**: Phase 3, separate. Request mbox export from Fastmail, then run `bundle exec rake import:mbox` against a one-shot Job that mounts the tarball.

---

## 6. Theming

The Bolt-styled theme is a separate repo (`thunderbird/discourse-theme-bolt`, planned). Themes are installed at runtime via Admin → Customize → Themes → "Install from Remote", pinned to a tag. We can also bake the theme git URL into the container as `DISCOURSE_THEMES_INSTALL=...` so a fresh deploy auto-installs.

Source of truth for design tokens: <https://bolt.thunderbird.net/8b179dbfd/p/038923-bolt-design-system>.

---

## 7. SSO

Phase 1 ships with Discourse's built-in user/password auth. Phase 1.5 swaps to OIDC against the Thundermail Keycloak realm using the bundled [`discourse-openid-connect`](https://github.com/discourse/discourse-openid-connect) plugin (already in the container build). The flip is config-only: add `DISCOURSE_OPENID_CONNECT_*` env vars and a Keycloak client.
