# Discourse — EKS Deployment

This repo holds the ArgoCD/ACK manifests and container build that run **[Discourse](https://www.discourse.org/)** self-hosted on the **`mzla-eks-workloads01`** EKS cluster (`eu-central-1`, account `668807881758`). The forum is exposed at **<https://discourse.thunderbird.net>** via Cloudflare Tunnel.

It is the planned replacement for Topicbox. Data migration from Topicbox is a later phase.

> **Audience:** SREs landing here for the first time should read *Architecture* + *Where things live* to understand the system. Senior SREs should jump to *Operational notes* + *Gotchas* for the things that bit us during initial deployment.

**Source of truth.** This README owns how to build and run the deployment, plus the bring-up runbook. Cross-repo wiring, estate context and dated findings about the *running* system live in [`docs/discourse.md`](https://github.com/thunderbird/platform-infrastructure/blob/main/docs/discourse.md); authentication lives in [`docs/discourse-auth.md`](https://github.com/thunderbird/platform-infrastructure/blob/main/docs/discourse-auth.md). Where this README and the running system disagree, **the running system wins** — settings here describe the intended design.

Build history: [#280](https://github.com/thunderbird/platform-infrastructure/pull/280) (initial Pulumi), [#285](https://github.com/thunderbird/platform-infrastructure/pull/285) (ECR + GHA OIDC), [#287](https://github.com/thunderbird/platform-infrastructure/pull/287) (YACE IRSA). The original epic [#279](https://github.com/thunderbird/platform-infrastructure/issues/279) is **closed with phases 2 (Bolt re-theme) and 3 (Topicbox import) never completed** — do not read it as a statement of current state.

---

## 1. What is Discourse?

Discourse is a Rails (Ruby 3.4) forum platform. Upstream officially supports Docker via [`discourse_docker`](https://github.com/discourse/discourse_docker) but [does not officially support Kubernetes](https://meta.discourse.org/t/installing-on-kubernetes/49329). The image we ship is built by `discourse_docker`'s `launcher bootstrap` (which compiles assets, installs gems, and bakes Postgres + Redis binaries that go unused at runtime since we point at external services).

### Components we run

| Component | Image | Purpose |
|-----------|-------|---------|
| `discourse-web` | `668807881758.dkr.ecr.eu-central-1.amazonaws.com/discourse:latest` | nginx → Puma + Rails on `:80`. 2 replicas. discourse-prometheus collector on `:9405` (pod-network only). |
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
│   │   ├── discourse-web.yaml                      Deployment + Service for nginx+Puma (image `:latest`)
│   │   ├── discourse-sidekiq.yaml                  Deployment for Sidekiq workers (image `:latest`)
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
11. **Re-enter the authentication config.** Auth is database state, so a fresh site comes up with **no OAuth providers and no client credentials** — see [Authentication](#7-authentication). This step cannot be skipped for a DR rebuild: local login alone will not let existing users back in.

---

## 5. Operational notes

### Image upgrades

The `image:` field in `argocd/workloads/discourse-{web,sidekiq,migrate-job}.yaml` tracks `:latest` with `imagePullPolicy: Always`. Because the manifest doesn't change between builds, ArgoCD won't roll anything on its own, and any pod restart (node drain, eviction, crash) pulls whatever `:latest` currently points at. To roll a new image:

1. Tag this repo `v0.1.x` — GHA builds and pushes `:v0.1.x` and `:latest`
2. Run migrations: trigger an ArgoCD sync of the discourse app (the `discourse-db-migrate` Sync hook re-runs on every sync)
3. Restart the workloads: `kubectl rollout restart deploy/discourse-web deploy/discourse-sidekiq -n discourse`

To roll back, pin the three manifests to a known-good `v0.1.x` tag.

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

## 6. Theming

The Bolt-styled theme lives in [thunderbird/discourse-theme-bolt](https://github.com/thunderbird/discourse-theme-bolt) (public; v0.1.0). Install via Admin UI → Customize → Themes → "Install from a git repository". Theme covers basic palette + Inter font; full Bolt palette + dark scheme tracked in that repo's TODO.

---

## 7. Authentication

Restores the section dropped in PR #15, rewritten to what actually shipped. The full runbook — break-glass, rollback, account matching, and why Discourse ID was rejected — is [`docs/discourse-auth.md`](https://github.com/thunderbird/platform-infrastructure/blob/main/docs/discourse-auth.md) in `platform-infrastructure`. This section covers only what someone rebuilding *this* deployment needs.

### What is enabled

| Method | Setting | Where the credentials are |
|---|---|---|
| GitHub OAuth | `enable_github_logins` | OAuth App under the **`thunderbird` GitHub org** (Settings → Developer settings → OAuth Apps). Callback `https://discourse.thunderbird.net/auth/github/callback` |
| Google OAuth | `enable_google_oauth2_logins` | GCP project **`thunderbird-discourse`** in the mozilla.com org (`1047413191847`). Callback `https://discourse.thunderbird.net/auth/google_oauth2/callback` |
| Local password | `enable_local_logins` | Discourse's own user table. **Keep this on** — it is the break-glass path |
| Email magic link | `enable_local_logins_via_email` | Rides SES (`eu-central-1`, `noreply@thunderbird.net`) |

**Discourse ID is not used** (`enable_discourse_id` false) and neither is DiscourseConnect. The earlier plan to swap to OIDC against a Keycloak realm did not happen either.

### None of this is in git, and that is not fixable here

Discourse stores auth settings — **including both OAuth client secrets** — as rows in the RDS `site_settings` table. `enable_*_logins` has no entry in `config/discourse_defaults.conf`, so there is **no `DISCOURSE_*` env var for it**: adding one to `discourse-config.yaml` is a silent no-op that ArgoCD still reports as Synced and Healthy. Do not try.

Consequences:

- **A fresh cluster or DR rebuild comes up with authentication unconfigured** (runbook step 11). The live values exist only inside the RDS snapshot.
- **Never commit the client IDs or secrets to this repo.** It is public-facing config; the secrets belong in the admin UI only.
- Rotation is two-sided and manual: rotate at the provider, then set the new value in Discourse. Nothing syncs them, and nothing alerts when they diverge — the symptom is every login through that provider failing at once.

### Setting or restoring it

Admin → Login & authentication, or from a Rails console in a web pod:

```
POD=$(kubectl get pod -n discourse -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it -n discourse "$POD" -- bash -lc 'cd /var/www/discourse && su discourse -c "bundle exec rails c"'
```

```ruby
SiteSetting.github_client_id = "..."
SiteSetting.github_client_secret = "..."
SiteSetting.enable_github_logins = true
SiteSetting.google_oauth2_client_id = "..."
SiteSetting.google_oauth2_client_secret = "..."
SiteSetting.enable_google_oauth2_logins = true
```

### If nobody can log in

1. Local password login at `/login` — always on, independent of both providers. Note an account created *through* GitHub or Google has no password until it resets one, and reset mail needs SES.
2. `/u/admin-login` magic link (SES again).
3. The Rails console above; or `RAILS_ENV=production bundle exec rake admin:create` in a web pod. Discourse's documented `./launcher enter app` rescue **does not exist here** — there is no launcher at runtime.
4. Never leave the site with zero login methods enabled; Discourse's "No login methods are configured" state is recoverable only from that console.
