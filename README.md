# GCP RSS TL;DR

A serverless bot that fetches GCP release notes from RSS/Atom, summarizes them via OpenRouter,
and posts to Telegram. GCP sibling of [aws-rss-tldr](https://github.com/jonatan5524/aws-rss-tldr)
— same shape, GCP-native services.

## Status

Design-only. Nothing is built yet. This README and `CLAUDE.md` capture the architecture
decisions made before any code was written, so a fresh session can start implementing directly.

## Architecture

| Concern              | AWS version (`aws-rss-tldr`)        | This version (GCP)                                   |
|-----------------------|--------------------------------------|--------------------------------------------------------|
| Compute               | Lambda                               | Cloud Run Jobs                                        |
| Trigger               | EventBridge scheduled rule           | Cloud Scheduler → HTTP `:run` on the Job (OIDC)        |
| Secrets               | SSM Parameter Store, fetched via `boto3` in code | Secret Manager, injected natively as env vars (no SDK call) |
| Runtime identity      | IAM role (`ssm:GetParameter`, `kms:Decrypt`) | Dedicated service account (`secretmanager.secretAccessor` on 3 secrets only) |
| Deploy identity       | GitHub Actions OIDC → IAM role       | GitHub Actions Workload Identity Federation → deploy service account |
| Image/package build   | `null_resource`/`local-exec` Docker build inside Terraform, zip artifact | GitHub Actions builds & pushes container image, tagged by git SHA; Terraform only references the tag |
| Terraform state       | S3 bucket                            | GCS bucket                                            |
| Region                | `il-central-1`                       | `me-west1`                                            |
| Retries               | Exception re-raised → EventBridge retries the Lambda invocation | Cloud Run Job `max_retries`                           |
| Resource sizing        | 256 MB memory, 600s timeout          | 1 vCPU, 256Mi memory, ~600s task timeout (parity)      |
| Base image             | AWS Lambda Python 3.12 runtime image | `python:3.12-slim`                                    |
| State store for dedup  | **None** — stateless, filters RSS entries published in the last 24h on each run | Same: stateless, no datastore |
| Source feed            | `https://aws.amazon.com/about-aws/whats-new/recent/feed/` | `https://docs.cloud.google.com/feeds/gcp-release-notes.xml` |
| Local dev              | `local_test.py` mocks `get_secret()` (SSM fetch) | Not needed — secrets are plain env vars, just `export` and run |
| GCP project            | n/a                                   | New, dedicated project                                 |
| CI workflow shape      | Single job: checkout → auth → `terraform plan`/`apply` | Single job: checkout → WIF auth → build & push image → `terraform plan`/`apply` referencing the new tag |

### Why this shape

- **Cloud Run Jobs, not Cloud Functions or a Cloud Run service**: this is a scheduled,
  run-to-completion batch task with no HTTP response to give. Jobs matches that directly;
  Functions/a Run service are built around request/response semantics and would need a fake
  HTTP wrapper around a task that returns nothing meaningful.
- **Cloud Scheduler → HTTP `:run` with OIDC**: there is no more direct native trigger for a
  Cloud Run Job (no built-in cron field on the resource itself, unlike a Kubernetes CronJob).
  This is Google's documented pattern for scheduling a Job.
- **No datastore**: the AWS version's README mentions DynamoDB but none exists in the actual
  Terraform/code — it's stateless, filtering the feed to the last 24h on every run. The GCP
  version inherits that; no dedup/state store to design or port.
- **Secrets created manually, Terraform only references names**: mirrors the AWS SSM setup —
  secret *values* never touch Terraform files or state.
- **Native secret injection over an SDK call**: Cloud Run Jobs can mount Secret Manager secrets
  directly as env vars at deploy time, so the container just reads `os.environ[...]` — simpler
  than the AWS version's dedicated `get_secret()`/`boto3` wrapper, and the idiomatic Cloud Run
  pattern.
- **Separate deploy vs. runtime service accounts**: the CI/deploy identity needs broad
  permissions (Run admin, Artifact Registry write, GCS state access); the Job's runtime identity
  needs only `secretmanager.secretAccessor` on three specific secrets. Collapsing them would let
  the running container's identity redeploy itself — unnecessary blast radius.
- **Workload Identity Federation over a service-account JSON key**: no long-lived credential
  stored in GitHub secrets; matches the AWS version, which already uses OIDC
  (`aws-actions/configure-aws-credentials` with `role-to-assume`), not a static key.
- **Image build in GitHub Actions, not Terraform**: Cloud Run Jobs needs an image already in
  Artifact Registry before Terraform can reference it — unlike Lambda's zip, which Terraform
  builds inline via `local-exec`. Keeping Terraform purely declarative (build elsewhere, deploy
  references a tag) avoids baking Docker calls into `.tf` files.
- **Git-SHA image tags, not `latest`**: `latest` leaves Terraform blind to image changes (the Job
  resource's image string never changes, so `apply` won't know to redeploy) and makes rollback
  guesswork. SHA tags give precise redeploys and trivial rollback.
- **New dedicated GCP project**: isolates free-tier accounting, IAM, and teardown blast radius
  from any other GCP work. Note: Cloud Scheduler's "3 free jobs/month" allowance is per *billing
  account*, not per project — check that pool isn't already used elsewhere on the same billing
  account.
- **`me-west1`**: closest GCP region to AWS's `il-central-1` (both are Tel Aviv-area regions).

### Free tier

Expected to run comfortably within GCP's Always Free tier: one Cloud Run Job execution/day is a
tiny fraction of the monthly vCPU-second/GiB-second allowance; Cloud Scheduler needs only 1 of
the 3 free jobs/month (per billing account); Secret Manager access volume (3 secrets, read once
daily) is trivial. Exact numeric thresholds weren't independently re-verified against the current
pricing pages during design — confirm before relying on them long-term.

## Prerequisites (anticipated)

- GCP project (new, dedicated) with billing enabled
- Terraform >= 1.6.0
- Telegram Bot Token (from [@BotFather](https://t.me/botfather))
- OpenRouter API Key (from [openrouter.ai](https://openrouter.ai/keys))
- GCS bucket for Terraform state (created once, manually — mirrors the AWS version's S3 step)
- Secret Manager secrets created once, manually (`telegram_bot_token`, `telegram_chat_id`,
  `openrouter_api_key`)
- Workload Identity Federation pool/provider configured for GitHub Actions

## License

MIT
