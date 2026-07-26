# AGENTS.md

Pure-Harn connector package for CircleCI Cloud (REST API v2 + inbound webhooks).

Shared connector authoring rules live in the Harn guide:

- [Connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md)

Put shared connector guidance in the Harn guide and keep only
provider-specific notes and local hazards here.

`CLAUDE.md` points here. Edit `AGENTS.md` only.

## Provider notes

- `circleci-signature` is a comma-separated, versioned list (`v1=<hex>`). `v1` is the HMAC-SHA256
  hex digest of the **raw request body** keyed by the configured webhook signing secret. Recompute
  and compare with `constant_time_eq`; verify only the highest known version (downgrade defense).
- There is no timestamp in the signature, so there is no replay window. Dedup on the top-level `id`
  delivery UUID instead — CircleCI reuses it on retries.
- Webhook body `type` is `workflow-completed` or `job-completed`; the `circleci-event-type` header
  echoes it. A run is a failure when `status` is `failed` or `error` (not `canceled`/`unauthorized`).
- Payloads carry no reliable PR field. Resolve PRs out-of-band by SHA via the optional
  `github_commit_pulls` helper (GitHub `/commits/{sha}/pulls`).
- Outbound REST v2 auth is the `Circle-Token` header (not bearer). Mutating methods
  (`workflow.rerun`, `workflow.cancel`) are flagged `requires_approval` in `methods()`.
- Do not add compatibility shims or deprecation aliases in this nascent package; cut over directly
  when behavior changes.

<!-- BEGIN HARN SHARED AGENT CONTRACT: managed by harn-bump-fleet -->

## Ecosystem working agreement

- Pursue the ambitious product outcome; make the seams boring with small typed
  interfaces, explicit invariants, and deterministic projections.
- Give each behavior one semantic owner. Generate or parity-test other surfaces
  instead of maintaining competing implementations.
- Work autonomously inside approved scope. Pause for destructive, production,
  high-spend, ambiguous, or authority-expanding actions—not routine reversible work.
- Treat stop, wait, stand down, and pivot as control events for long-lived work.
- Match evidence to the claim: exercise the canonical user path, state the
  falsifier, verify liveness and recovery, and record residual blind spots.
- "Ship" means landed on main with required deploy and post-merge checks complete.

<!-- END HARN SHARED AGENT CONTRACT -->
