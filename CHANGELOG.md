# Changelog

All notable changes to `harn-circleci-connector` will be documented in
this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Connector contract v2 with product-facing service metadata: the
  manifest now declares all seven outbound CircleCI operations
  (`workflow.get`, `workflow.jobs`, `job.get`, `artifacts.list`,
  `workflow.rerun`, `workflow.cancel`, `api.request`) with their
  capability, purpose, effect, evidence, and redaction semantics.
  `api.request` is declared consequential because the caller chooses the
  verb, so it cannot be projected as read-only.
- Initial pure-Harn CircleCI connector (connector interface, payload
  schema, lifecycle functions).
- `normalize_inbound` with HMAC-SHA256 signature verification against the
  versioned `circleci-signature` header (`v1=<hex>` over the raw body),
  constant-time comparison, highest-version-only (downgrade defense), and
  fail-closed behavior when no signing secret is configured.
- Normalized CI envelope for `workflow-completed` and `job-completed`
  events: `provider`, `kind`, `status`, `is_failure`, `repo`,
  `commit_sha`, `branch`, `project_slug`, `workflow_id`,
  `pipeline_number`, `web_url`, optional `job`, `rerun_handle`, and a
  `triage` shape for failure-handling triggers.
- Delivery-`id` deduplication (CircleCI reuses the id on retries) in
  place of a replay window, since the signature carries no timestamp.
- Outbound `call(method, args)` dispatch over CircleCI REST API v2
  (`Circle-Token` auth): `workflow.rerun` (from-failed or explicit jobs),
  `workflow.cancel`, `workflow.get`, `workflow.jobs`, `job.get`,
  `artifacts.list`, and an `api.request` escape hatch.
- `methods()` metadata flagging mutating methods (`workflow.rerun`,
  `workflow.cancel`) as `requires_approval`.
- Optional `github_commit_pulls(owner, repo, sha, github_token)` helper to
  resolve PRs by commit SHA, since CircleCI payloads carry no PR field.
- Rate-limit-aware retry (`X-RateLimit-Remaining` / `Retry-After`) with a
  60-second sleep cap, and an https-only egress allowlist that blocks
  loopback/private/link-local hosts.
- Smoke tests and connector-contract fixtures covering normalization,
  signature pass/fail, fail-closed, dedup, and outbound dispatch.

[Unreleased]: https://github.com/burin-labs/harn-circleci-connector/compare/main
