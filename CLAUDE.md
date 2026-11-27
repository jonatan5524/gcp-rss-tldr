# gcp-rss-tldr — working notes for Claude

This repo is the GCP port of `~/git/aws-rss-tldr`. The design was settled in a `/grilling`
session before any code existed — read `README.md` first, it has the full architecture table
and rationale. This file is about *how to build it*, not what was decided.

## Starting point

- Nothing is built yet: no Terraform, no Dockerfile, no handler code, no GitHub Actions workflow.
- The sibling repo `~/git/aws-rss-tldr` is the reference implementation. Port its logic
  (`lambda/handler.py`'s RSS fetch → OpenRouter summarize → Telegram send flow), not its
  AWS-specific plumbing (no `boto3`, no SSM `get_secret()` wrapper — secrets arrive as plain env
  vars via Cloud Run's native Secret Manager injection, see README).
- The OpenRouter prompt needs rewriting for GCP release notes instead of AWS "What's New"
  (service/feature naming, "before/now" framing should still make sense for GCP announcements —
  check what fields the GCP feed's entries actually contain before assuming parity with AWS's).

## Settled decisions (don't re-litigate without a reason)

See README's architecture table for the full list. Highlights most likely to trip up
implementation:

- **Secrets**: created manually via `gcloud`, Terraform only grants `secretAccessor` by name —
  never put a secret value in a `.tf` file or in state.
- **Image build lives in GitHub Actions**, not Terraform. Terraform's Cloud Run Job resource
  takes an image reference as a variable (git SHA tag) passed in from CI — don't add a
  `local-exec`/`null_resource` Docker build like the AWS repo's `main.tf` has for the Lambda zip.
- **Two service accounts**: one for CI/deploy (via WIF, broad permissions), one for the Job's
  runtime identity (secret-accessor only, nothing else). Don't collapse these.
- **No datastore anywhere.** The AWS repo's README mentions DynamoDB but none exists in its
  actual code — don't add one here either. Dedup is "filter feed entries from the last 24h,"
  full stop.
- **Region `me-west1`, new dedicated GCP project**, GCS backend for Terraform state (bucket
  created manually once, like the AWS repo's S3 step).

## Open items to verify once you have gcloud access

These were flagged during design as not independently confirmed — check before relying on them:

- Exact Cloud Run Jobs / Secret Manager / Cloud Scheduler Always Free numeric thresholds.
- Whether `me-west1` is enabled on the billing account in use; fall back to `europe-west3` or
  `europe-west8` if not.
- The actual structure of entries in `https://docs.cloud.google.com/feeds/gcp-release-notes.xml`
  (Atom, not RSS 2.0 — `feedparser` handles both, but confirm field names like `summary`,
  `published_parsed` are populated the way `handler.py` expects before porting the parsing logic
  as-is).

## Suggested build order

1. `gcloud` project setup + GCS state bucket + Secret Manager secrets (manual, one-time, document
   the exact commands in README's Setup section as you go).
2. Terraform: provider config, service accounts + IAM bindings, Artifact Registry repo, Cloud Run
   Job resource (referencing a placeholder/bootstrap image tag), Cloud Scheduler job, WIF
   pool/provider.
3. `Dockerfile` (`python:3.12-slim`) + ported handler code, tested locally via plain env vars
   (no mock wrapper needed — see README's "Local dev" row).
4. GitHub Actions workflow mirroring `aws-rss-tldr/.github/workflows/deploy.yaml`'s shape: single
   job, checkout → WIF auth → build & push (git SHA tag) → `terraform plan`/`apply`.
5. End-to-end dry run against a test Telegram chat before pointing at any real channel.

## Conventions carried over from aws-rss-tldr

- `.pre-commit-config.yaml`: same hooks (gitleaks, terraform fmt/validate, trailing-whitespace,
  etc.) minus the AWS-specific `detect-aws-credentials` hook.
- Keep env var names parallel to the AWS version where there's a direct equivalent
  (`PRIMARY_MODEL`, `FALLBACK_MODELS`) so the two repos stay easy to compare side by side.
