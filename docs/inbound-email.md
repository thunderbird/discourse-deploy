# Inbound email (reply-by-email) for discourse.thunderbird.net

Issue: [thunderbird/PE-Routing#20](https://github.com/thunderbird/PE-Routing/issues/20). Outbound mail (SES SMTP, eu-central-1, production access, 50k/day) already works. This doc covers inbound.

## Constraint

Discourse runs on `mzla-eks-workloads01` behind a Cloudflare Tunnel, with **zero inbound ports**. The public hostname also sits behind a Cloudflare **managed challenge**. The stock answer from upstream (the `mail-receiver` container listening on :25, plus an MX pointing at the host) doesn't fit as-is.

## Options

| | a) SES inbound -> S3 -> in-cluster bridge -> `handle_mail` | b) `mail-receiver` behind an NLB on :25 | c) POP3 polling (`pop3_polling_*`) |
|---|---|---|---|
| New inbound exposure | none (SES is the MX) | **new public :25 NLB**, our own spam/abuse surface | none |
| Spam/virus filtering | SES scan (`ScanEnabled`) plus Discourse's own checks | Postfix plus whatever we bolt on | the mailbox provider's |
| Moving parts | SES rule, S3 bucket, 1 tiny pod | NLB, EIP or DNS, mail-receiver Deployment, TLS cert for STARTTLS | a mailbox, its credentials, Sidekiq job |
| Addressing | any address `@discourse.thunderbird.net` (group/category addresses too) | same | **one mailbox**. Plus-addressing only works if the provider supports it, and group/category inbound addresses need catch-all aliasing |
| Latency | <= poll interval (30 s) | immediate | `pop3_polling_period_mins` (min 1, default 5) |
| Precedent here | **[k9forum](https://github.com/thunderbird/k9forum) already runs this** (SES us-east-2 -> S3 -> Lambda -> `handle_mail`) | none | none |
| Ops burden | low | highest (we'd run an MTA) | low, but tied to a human-style mailbox (password rotation, Google/Thundermail policy, deliverability of a shared box) |

**Recommendation: (a).** SES email receiving is supported in **eu-central-1** (`inbound-smtp.eu-central-1.amazonaws.com`, [AWS endpoints](https://docs.aws.amazon.com/general/latest/gr/ses.html#ses_inbound_endpoints)), the same region and account as outbound, so no cross-region pieces. It's the k9forum pattern with one change: k9forum uses a Lambda that POSTs to the public URL, which would hit the Cloudflare challenge here, so the bridge runs **inside the cluster** and calls the `discourse-web` Service directly. That also avoids a new Cloudflare WAF exception (brief A owns Cloudflare Access/WAF) and keeps credentials on IRSA.

(c) is the fallback if we don't want any SES receipt plumbing. It's fine for replies only, but weak for the Topicbox-replacement goal (group mail addresses per list, [#22](https://github.com/thunderbird/PE-Routing/issues/22)/[#23](https://github.com/thunderbird/PE-Routing/issues/23)). (b) adds the only new inbound port in this stack and makes us run an MTA. Rejected.

## Flow

```
sender MTA --MX--> inbound-smtp.eu-central-1.amazonaws.com  (SES, receipt rule set "discourse-inbound")
                      | rule "discourse-replies": recipients [discourse.thunderbird.net], ScanEnabled, TLS Require
                      v
           s3://mzla-discourse-inbound/incoming/<messageId>          (ACK Bucket, this repo)
                      | poll every 30 s (IRSA: workloads-prod-discourse-mail-bridge)
                      v
           discourse-mail-bridge pod (this repo, replicas: 0 until go-live)
                      | POST http://discourse-web/admin/email/handle_mail   email_encoded=<base64>
                      |   Host: discourse.thunderbird.net, Api-Key (scope email:receive_emails)
                      v
           Discourse enqueues Jobs::ProcessEmail -> Sidekiq -> post / PM / new topic
           object moved to processed/ (2xx) or failed/ (4xx, or >24 h of 5xx/429)
```

## What's in this repo ([discourse-deploy#20](https://github.com/thunderbird/discourse-deploy/pull/20))

- `argocd/aws-resources/s3-inbound-bucket.yaml`: ACK Bucket `mzla-discourse-inbound`. SSE-S3, public access blocked, lifecycle (incoming/processed 14 d, failed 60 d), and a bucket policy that only lets this account's `discourse-inbound` receipt rules `PutObject` into `incoming/`.
- `argocd/secrets/discourse-mail-bridge-credentials.yaml`: ExternalSecret `mzla/discourse/inbound` -> `DISCOURSE_API_KEY`.
- `argocd/workloads/discourse-mail-bridge.yaml`: ServiceAccount (IRSA), ConfigMap with the bridge script, and a Deployment at **`replicas: 0`**.

## What has to go elsewhere (not in this PR)

### platform-infrastructure Pulumi ([platform-infrastructure#1510](https://github.com/thunderbird/platform-infrastructure/pull/1510), `pulumi/environments/mzla-workloads`)
1. IRSA role next to `discourse-default`:
   ```yaml
       discourse-mail-bridge:
         namespace: discourse
         service_account: discourse-mail-bridge
         policies:
           - effect: Allow
             actions: [s3:ListBucket]
             resources: ["arn:aws:s3:::mzla-discourse-inbound"]
           - effect: Allow
             actions: [s3:GetObject, s3:PutObject, s3:DeleteObject]
             resources: ["arn:aws:s3:::mzla-discourse-inbound/*"]
   ```
2. Secrets Manager secret shell `mzla/discourse/inbound` (the value gets set by hand after the API key exists).
3. SES receipt rule set `discourse-inbound`, rule `discourse-replies`: recipients `discourse.thunderbird.net`, `ScanEnabled: true`, `TlsPolicy: Require`, one `S3Action` (bucket `mzla-discourse-inbound`, prefix `incoming/`), **then activate the rule set** (only one rule set can be active per account and region; workloads/eu-central-1 has none today). No `LambdaAction` (the k9forum rule has one that errors on every message).
4. SES receiving needs a verified identity for the recipient domain. `thunderbird.net` is a verified domain identity in this account and region. Confirm it covers the `discourse.` subdomain for receiving; if not, add a `discourse.thunderbird.net` domain identity (Easy DKIM CNAMEs). This touches only workloads-account SES, not Thundermail identities (contract with brief B2/#6).

Merge order: #1510 **first** (role and secret; its SES flags `receipt_rule`/`active` start `false`), then this PR. Otherwise the ExternalSecret shows `SecretSyncedError` and the Argo app goes Degraded. The receipt rule comes after this PR because SES test-writes the bucket when the rule is created.

### DNS (thunderbird.net zone, Cloudflare): a change, Matthew's go
| Name | Type | Value | Notes |
|---|---|---|---|
| `discourse.thunderbird.net` | MX | `10 inbound-smtp.eu-central-1.amazonaws.com` | Coexists with the proxied A/AAAA records (MX isn't proxied). Add it only at go-live: once it exists, mail starts landing in S3. |
| `discourse.thunderbird.net` | TXT | `v=spf1 -all` (optional) | Nothing sends *as* `@discourse.thunderbird.net`. Notifications go out as `noreply@thunderbird.net`, so they align with the existing thunderbird.net SPF/DKIM/DMARC through SES. Only needed if we later change the From domain. |

**SPF/DKIM/DMARC alignment:** outbound is unchanged. `DISCOURSE_NOTIFICATION_EMAIL=noreply@thunderbird.net`, sent via SES with the verified `thunderbird.net` identity (Easy DKIM, d=thunderbird.net), so DMARC aligns on DKIM. The reply address only shows up in `Reply-To`, which DMARC doesn't evaluate. If Discourse is later set to send group mail *from* `group@discourse.thunderbird.net`, that subdomain needs its own SES identity, DKIM, and a custom MAIL FROM first. Track that separately.

A dedicated subdomain (`reply.discourse.thunderbird.net`) works the same way if we'd rather keep MX off the web hostname. Swap the recipient, MX name, and `reply_by_email_address`.

### Discourse site settings (Admin -> Settings -> Email), at go-live
| Setting | Value |
|---|---|
| `reply_by_email_address` | `replies+%{reply_key}@discourse.thunderbird.net` |
| `manual_polling_enabled` | `true` (this is what enables the `handle_mail` API) |
| `reply_by_email_enabled` | `true` |
| `pop3_polling_enabled` | `false` |
| `email_in` | `true` only once group/category inbound addresses are wanted (#22/#23). Then set the per-category or per-group incoming address (e.g. `council@discourse.thunderbird.net`) |
| `email_in_min_trust` | leave at the default (2) unless the Topicbox migration decides otherwise |
| `find_related_post_with_key`, `strip_incoming_email_lines` | defaults |

API key: Admin -> API -> New -> user `system`, **Granular** scope `email: receive_emails` only. Put it in `mzla/discourse/inbound` as `{"api_key": "..."}`.

### Go-live sequence (each step is a live change and needs Matthew's go)
0. **Gate:** not while the Topicbox imports ([#22](https://github.com/thunderbird/PE-Routing/issues/22)/[#23](https://github.com/thunderbird/PE-Routing/issues/23)) are pending. Imports run with outbound email disabled, and inbound shouldn't be live either.
1. Merge [platform-infrastructure#1510](https://github.com/thunderbird/platform-infrastructure/pull/1510) (role and secret shell; SES flags off).
2. Merge this PR. Argo creates the bucket and the ExternalSecret; the bridge stays at 0 replicas. Then a small Pulumi PR sets `discourse_inbound.receipt_rule: true` (rule set and rule created, still inactive).
3. Create the API key and put it in SM. Set the site settings above.
4. Scale the bridge to 1 by PR (`replicas: 1`).
5. Pulumi `discourse_inbound.active: true` (activates the rule set), and add the MX.
6. Verify: send mail to `replies+test@discourse.thunderbird.net`. It should show up under `processed/` and in Admin -> Emails -> Received/Rejected. Then reply by email to a real notification and confirm the post appears. Check the bridge logs (`kubectl -n discourse logs deploy/discourse-mail-bridge`).

### Rollback
Remove the MX record (senders queue and retry, then bounce after their retry window) or deactivate the receipt rule set, and scale the bridge to 0. No data migration involved. Messages already in S3 stay until lifecycle expiry and can be replayed by moving them back into `incoming/`.

## Notes
- Merging anything to this repo's `main` triggers an ArgoCD auto-sync (`prune`, `selfHeal`), and every sync re-runs the `discourse-db-migrate` hook against RDS. Expected, but worth knowing when you merge.
- The bridge uses `public.ecr.aws/lambda/python:3.12` only because it ships boto3 on arm64. If that feels odd, a 10-line Dockerfile built by this repo's GHA into ECR is the alternative.
