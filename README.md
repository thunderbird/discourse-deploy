# Discourse — EKS Deployment

This repo holds the ArgoCD/ACK manifests and container build that run **[Discourse](https://www.discourse.org/)** self-hosted on the **`mzla-eks-workloads01`** EKS cluster (`eu-central-1`, account `668807881758`). The forum is exposed at **<https://discourse.thunderbird.net>** via Cloudflare Tunnel.

It is the planned replacement for Topicbox. Data migration from Topicbox is a later phase.

> **Audience:** SREs landing here for the first time should read *Architecture* + *Where things live* to understand the system. Senior SREs should jump to *Operational notes* + *Gotchas* for the things that bit us during initial deployment.

**Source of truth for the deployment plan and decision history:** [thunderbird/platform-infrastructure#279](https://github.com/thunderbird/platform-infrastructure/issues/279) (epic), [#280](https://github.com/thunderbird/platform-infrastructure/pull/280) (initial Pulumi), [#285](https://github.com/thunderbird/platform-infrastructure/pull/285) (ECR + GHA OIDC), [#287](https://github.com/thunderbird/platform-infrastructure/pull/287) (YACE IRSA).

---

## 1. What is Discourse?

Discourse is a Rails (Ruby 3.4) forum platform. Upstream officially supports Docker via [`discourse_docker`](https://github.com/discourse/discourse_docker) but [does not officially support Kubernetes](https://meta.discourse.org/t/installing-on-kubernetes/49329). The image we ship is built by `discourse_docker`'s `launcher bootstrap` (which compiles assets, installs gems, and bakes Postgres + Redis binaries that go unused at runtime since we point at external services).

### Components we run

| Component | Image | Purpose |
|-----------|-------|---------|
| `discourse-web` | `668807881758.dkr.ecr.eu-central-1.amazonaws.com/discourse:v0.1.2` | nginx → Puma + Rails on `:80`. 2 replicas. discourse-prometheus collector on `:9405` (pod-network only). |
| `discourse-sidekiq` | same image, `bundle exec sidekiq` | Background jobs (mail, search index, digest). Reports metrics to web's collector via IPC. 1 replica. |
| `discourse-tunnel` | `cloudflare/cloudflared` (managed by `cloudflare-operator`) | Outbound tunnel. Zero inbound ports. |
| `discourse-postgres` (RDS) | `postgres:16.6` (AWS RDS) | Application database. 7-day automated backup retention. |
| `discourse-redis` (ElastiCache) | `redis:7.1` (AWS ElastiCache) | Cache + Sidekiq queue. AUTH-token enabled, transit encryption on. |
| `mzla-discourse-uploads` (S3) | — | Avatars, attachments, post images, backups. Pod access via IRSA (no static keys). |
| SES (eu-central-1) | — | Outbound SMTP. Production-access enabled (50k/24h, 14/sec). |
| `yace` | `quay.io/prometheuscommunity/yet-another-cloudwatch-exporter:v0.64.0` | Pulls AWS service metrics for the dashboard. |

### Plugins baked into the image

Bundled by `discourse_docker` base — `openid-connect`, `chat`, `solved`, `presence`, `data-explorer`, plus 30+ others (see `Admin → Plugins`).

Cloned in by `container/containers/app.yml` (NOT bundled in the base):
- `discourse-prometheus` — exposes `/metrics` on `:9405` for VMAgent

---

## 2. Architecture

```
            Internet
                │
                ▼
  ┌───────────────────────────────┐
  │  Cloudflare edge (managed     │
  │  challenge + TLS termination  │
  │  on thunderbird.net zone)     │
  └───────────────────────────────┘
                │  outbound tunnel
                ▼
  ┌───────────────────────────────┐    namespace: discourse
  │  discourse-tunnel             │    on mzla-eks-workloads01 (eu-central-1)
  │  (cloudflared, 4 conns to     │
  │   fra03/fra14/fra16/fra08)    │
  └───────────────────────────────┘
                │  http://discourse-web:80 → nginx → unicorn(127.0.0.1:3000)
                ▼
  ┌───────────────────────────────┐
  │  discourse-web (nginx + Puma) │──── :9405/metrics ──► VMAgent
  │  - HTTP API + UI              │      (only after PROMETHEUS_WEBSERVER_BIND=0.0.0.0)
  │  - discourse-prometheus       │
  │  - aggregates sidekiq metrics │
  │    via Unix socket IPC        │
  └───────────────────────────────┘
            │            │
            │            └──► s3:mzla-discourse-uploads (IRSA: workloads-prod-discourse-default)
            │
   ┌────────┴────────┐
   ▼                 ▼
┌──────────────┐    ┌──────────────────┐
│ ElastiCache  │    │  RDS Postgres    │
│ Redis 7.1    │    │  discourse-      │
│ (AUTH+TLS)   │    │  postgres        │
│ master.disc..│    │  16.6, db.t4g.   │
└──────────────┘    │  medium, 30Gi    │
        ▲           │  gp3, encrypted  │
        │ jobs      │  7d backups      │
        │           └──────────────────┘
   ┌────┴────────────────┐    ▲
   │ discourse-sidekiq   │────┘
   │ (bundle exec sidekiq│   reports metrics → web's collector via Unix socket
   └─────────────────────┘
```

YACE runs in the same namespace, scrapes CloudWatch via IRSA, exposes Prometheus metrics on `:5000` for VMAgent.

---

## 3. Where things live

### In this repo

```
discourse-deploy/
├── README.md                                       you are here
├── .github/
│   ├── CODEOWNERS                                  @thunderbird/platform-infrastructure
│   └── workflows/build.yml                         build + push to ECR on tag
├── container/
│   ├── Dockerfile                                  unused (placeholder; we use launcher bootstrap)
│   └── containers/app.yml                          discourse_docker config (templates, plugin clones)
├── argocd/
│   ├── aws-resources/
│   │   ├── rds-postgres.yaml                       ACK DBSubnetGroup + DBInstance (postgres 16.6)
│   │   ├── elasticache-redis.yaml                  ACK CacheSubnetGroup + ReplicationGroup (redis 7.1)
│   │   └── s3-bucket.yaml                          ACK Bucket: mzla-discourse-uploads
│   ├── secrets/
│   │   ├── discourse-db-credentials.yaml           ES → mzla/discourse/db
│   │   ├── discourse-redis-credentials.yaml        ES → mzla/discourse/redis
│   │   ├── discourse-app-secrets.yaml              ES → mzla/discourse/app
│   │   ├── discourse-smtp-credentials.yaml         ES → mzla/discourse/smtp
│   │   └── cloudflare-credentials.yaml             ES → mzla/twenty/cloudflare (shared)
│   ├── workloads/
│   │   ├── default-serviceaccount.yaml             IRSA annotation overlay (S3 access)
│   │   ├── rds-bootstrap-job.yaml                  CREATE USER + GRANTs + hstore + pg_trgm + vector
│   │   ├── discourse-config.yaml                   ConfigMap of DISCOURSE_* env
│   │   ├── discourse-web.yaml                      Deployment + Service for nginx+Puma (image pinned)
│   │   ├── discourse-sidekiq.yaml                  Deployment for Sidekiq workers (image pinned)
│   │   ├── discourse-migrate-job.yaml              Sync-hook Job: bundle exec rake db:migrate
│   │   └── cloudflare-tunnel.yaml                  Tunnel + TunnelBinding (discourse.thunderbird.net)
│   └── observability/
│       ├── discourse-vmpodscrape.yaml              VMPodScrape on web pods only (port 9405)
│       └── yace.yaml                               YACE Deployment + ConfigMap + scrape (port 5000)
```

### In `thunderbird/platform-infrastructure`

- `argocd/projects/discourse.yaml` — AppProject scoping discourse to workloads01
- `argocd/workloads/apps/discourse-app-of-apps.yaml` — sync pointer at this repo's `argocd/`
- `pulumi/environments/mzla-workloads/config.prod.yaml` — defines IRSA roles `workloads-prod-discourse-default` (S3), `workloads-prod-yace-discourse` (CloudWatch), `workloads-prod-discourse-deploy` (GHA OIDC for ECR push), the `workloads-prod-discourse-ses` IAM user (SES SMTP), and the four AWS Secrets Manager secrets at `mzla/discourse/{db,redis,app,smtp}`. Plus the ECR repo `discourse` and GitHub OIDC provider.

### In `thunderbird/discourse-theme-bolt`

The Bolt-styled Discourse theme. Public; v0.1.0 covers basic palette + Inter font. Installed via Admin → Customize → Themes → Install from a git repository → tag `v0.1.0`.

### In `thunderbird/platform-grafana`

`terraform/dashboards/discourse/overview.json` — Grafana dashboard at <https://grafana.pi.thunderbird.net/d/discourse-overview>. Datasource UID `P4169E866C3094E38` (the workloads01 VictoriaMetrics).

---

## 4. Bring-up runbook (for a fresh cluster or DR)

Order matters because the ACK CRDs report endpoints in their `status` only **after** AWS reconciles them; Discourse needs those endpoints in `discourse-config` and `rds-bootstrap-job` to come up.

1. **Pulumi up** in `mzla-workloads` to create IRSA, SES IAM user, and the four SM secrets.
2. **Provision ECR repo + GHA OIDC role** (also via Pulumi). Create the `production` GitHub Environment in this repo: `gh api -X PUT repos/thunderbird/discourse-deploy/environments/production`.
3. **Build the container image** by tagging this repo `v0.1.x`. The GHA workflow runs `launcher bootstrap` on `ubuntu-24.04-arm` (so the resulting image is arm64 — workloads01's default node groups are Graviton). Image is pushed to ECR as `:v0.1.x` and `:latest`.
4. **Register the app** by merging `argocd/projects/discourse.yaml` and `argocd/workloads/apps/discourse-app-of-apps.yaml` in `platform-infrastructure`. ArgoCD picks up this repo and starts syncing.
5. **Patch `<TBD>` placeholders** once RDS + ElastiCache are `available` (~10–15 min for RDS, ~5 min for Redis):
   ```
   kubectl get dbinstance discourse-postgres -n discourse \
     -o jsonpath='{.status.endpoint.address}'
   kubectl get replicationgroup discourse-redis -n discourse \
     -o jsonpath='{.status.nodeGroups[0].primaryEndpoint.address}'
   ```
   PR the resolved values into `discourse-config.yaml` (`DISCOURSE_DB_HOST`, `DISCOURSE_REDIS_HOST`) and `rds-bootstrap-job.yaml` (`PGHOST`).
6. **Wave 4 hooks** (`rds-bootstrap-job` + `discourse-migrate-job`) run after the patch syncs. Bootstrap creates `discourse_app_user` + grants + extensions (`hstore`, `pg_trgm`, `vector`); migrate runs all Discourse Rails migrations against RDS.
7. **Wave 5** brings up `discourse-web`, `discourse-sidekiq`, the Cloudflare tunnel, and the VMPodScrape. First Discourse boot takes ~3–5 min.
8. **Smoke**: `curl https://discourse.thunderbird.net/` (Cloudflare may challenge — open in browser); admin login uses creds from `mzla/discourse/app`.

   ```
   aws secretsmanager get-secret-value --secret-id mzla/discourse/app \
     --profile mzla-workloads --region eu-central-1 --query SecretString --output text | jq .
   ```
9. **Enable discourse-prometheus**: Admin → Plugins → discourse-prometheus → toggle on. Pod `:9405` starts serving metrics; VMAgent scrapes within 30s.
10. **Install the Bolt theme**: Admin → Customize → Themes → Install from a git repository → `https://github.com/thunderbird/discourse-theme-bolt`, tag `v0.1.0`. Set as default.

---

## 5. Operational notes

### Image upgrades

The `image:` field in `argocd/workloads/discourse-{web,sidekiq,migrate-job}.yaml` is pinned to a specific `v0.1.x` tag (not `:latest`) so a tag-triggered build doesn't auto-replace running pods on restart. To roll a new image:

1. Tag this repo `v0.1.x` — GHA builds and pushes
2. PR a manifest bump (`v0.1.{x-1}` → `v0.1.x`) in all three places
3. ArgoCD rolls

### Adding admins

Two paths:
- **Promote via UI**: user signs up at `/signup` → admin goes to `/admin/users` → search → "Grant Admin"
- **Auto-promote**: add their email to the `developerEmails` field in `mzla/discourse/app` (comma-separated). Restart web pods to pick up the new env: `kubectl rollout restart deploy/discourse-web -n discourse`

### Restarting pods

```
kubectl --context arn:aws:eks:eu-central-1:668807881758:cluster/mzla-eks-workloads01 \
  rollout restart deploy/discourse-web deploy/discourse-sidekiq -n discourse
```

### Backups

- RDS: 7-day automated retention, daily snapshot at 03:00 UTC. To take a manual snapshot:
  ```
  aws rds create-db-snapshot --db-instance-identifier discourse-postgres \
    --db-snapshot-identifier discourse-manual-$(date +%Y%m%d) \
    --profile mzla-workloads --region eu-central-1
  ```
- S3 uploads: bucket has versioning enabled; restore via `aws s3api list-object-versions` + `restore-object`
- Discourse-side: Admin → Backups → "Backup" creates a tarball in the S3 uploads bucket under `/backups/`

### Logs

- **CloudWatch**: log group `/eks/mzla/mzla-eks-workloads01/applications`, stream pattern `discourse/<pod>/<container>` (3-day retention). Vector ships there in addition to VictoriaLogs.
- **VictoriaLogs**: at https://grafana.pi.thunderbird.net (LogsQL datasource), filter `_stream:{namespace="discourse"}`

### Metrics + dashboards

- Grafana: <https://grafana.pi.thunderbird.net/d/discourse-overview>
- Discourse app metrics from discourse-prometheus on web pods `:9405`
- AWS metrics from YACE pod `:5000` (filtered by `project=discourse` tag)

---

## 6. Gotchas

These bit us during initial bring-up. Read before re-deploying or rebuilding.

- **Image is arm64-only.** workloads01's default node groups are Graviton; the amd64 nodegroup has `arch=amd64:NoSchedule`. The GHA build runs on `ubuntu-24.04-arm` so the image matches. Never `docker tag` an x86 image as `discourse:latest` — it'll silently land but pods won't schedule.
- **`launcher bootstrap` requires sudo + privileged Docker; `docker push` does NOT.** ECR credentials are configured for the runner user via `amazon-ecr-login`; `sudo docker push` uses root's empty docker config and 401s. Only the bootstrap step uses sudo.
- **`discourse_docker` requires `templates/postgres.template.yml` and `templates/redis.template.yml` to bootstrap.** Both are needed for `db:migrate` and `assets:precompile` during build, even though we don't use the bundled DB/Redis at runtime. Stripping them breaks bootstrap.
- **Two-phase deploy.** ACK doesn't surface RDS/Redis endpoints until *after* the resources reconcile, so the first sync ships `<TBD>` placeholders. A follow-up PR (after waiting ~15 min) drops the resolved values in. See bring-up step 5.
- **Cloudflare credentials shared with twenty.** ExternalSecret `cloudflare-secrets` references `mzla/twenty/cloudflare` (one CF account covers both apps). Don't create `mzla/discourse/cloudflare`.
- **Sidekiq image == web image.** Same OCI image, different `command:`. Don't fork or rebuild — let the Dockerfile produce one artifact and override entrypoint at the pod level.
- **Postgres extensions** `hstore`, `pg_trgm`, `vector` (pgvector for AI features) must be created as the master user; the app user lacks `rds_superuser`. `rds-bootstrap-job` handles this.
- **`discourse-prometheus` must be enabled AND bind-overridden.** The plugin defaults to `localhost:9405`. We bind to `0.0.0.0` via `DISCOURSE_PROMETHEUS_WEBSERVER_BIND=0.0.0.0` in the ConfigMap. Until the plugin is enabled in `Admin → Plugins`, the port doesn't open at all.
- **Sidekiq pods don't serve `:9405`.** The collector runs on web pods only; sidekiq processes report via IPC to the web collector. Scraping sidekiq pods just creates a permanently-down target. The VMPodScrape selector matches `component=web` only.
- **psql `-c "... :'app_pw'"` doesn't process variable substitution.** The `:'name'` syntax only works when SQL is read from stdin/`-f`. The bootstrap job uses a heredoc.
- **`DISCOURSE_SECRET_KEY_BASE` must be 128 hex chars.** The validation regex is `/\A[0-9a-f]{128}\z/`. Pulumi's `RandomPassword(special=False)` produces alphanumeric (a-z, A-Z, 0-9) which **fails** that validation — Discourse logs `WARNING: DISCOURSE_SECRET_KEY_BASE is invalid, it was re-generated` and uses an in-memory random instead. Real-world impact: sessions invalidate on pod restart. Future fix: use a hex-only random generator in Pulumi.
- **Topicbox import**: Phase 3, separate. Request mbox export from Fastmail, then run `bundle exec rake import:mbox` against a Job mounting the tarball.

---

## 7. Theming

The Bolt-styled theme lives in [thunderbird/discourse-theme-bolt](https://github.com/thunderbird/discourse-theme-bolt) (public; v0.1.0). Install via Admin UI → Customize → Themes → "Install from a git repository". Theme covers basic palette + Inter font; full Bolt palette + dark scheme tracked in that repo's TODO.

---

## 8. SSO

Phase 1 (current) uses Discourse's built-in user/password. Phase 1.5 (planned) swaps to OIDC against the Thundermail Keycloak realm using the bundled `discourse-openid-connect` plugin — config-only flip:

- Create OIDC client in Keycloak `thunderbird` realm
- Add `mzla/discourse/oidc` SM secret with client id + client secret
- Add `DISCOURSE_OPENID_CONNECT_*` env to ConfigMap

Tracked in M5 of the migration epic.
