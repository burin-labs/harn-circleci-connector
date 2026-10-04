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

## Pull request titles

Use `[Area] Sentence case`. The area is one of `Connector`, `CI`, or `Docs`.

- `[Connector] Reject webhook deliveries with a stale timestamp`
- `[CI] Repin the shared Harn package workflow`
- `[Docs] Describe the poll cursor contract`

Keep the title on one line, under about 70 characters. Say what changed, not
which files moved. Capitalize the first word after the bracket and leave the
rest in sentence case.

`CONTRIBUTING.md` states the contribution policy for this repository.

<!-- BEGIN HARN SHARED AGENT CONTRACT: managed by harn-bump-fleet -->

## Ecosystem working agreement

- Pursue the ambitious product outcome; make the seams boring with small typed
  interfaces, explicit invariants, and deterministic projections.
- Give each behavior one semantic owner. Generate or parity-test other surfaces
  instead of maintaining competing implementations.
- Work autonomously inside approved scope. Pause for destructive, production,
  high-spend, ambiguous, or authority-expanding actions—not routine reversible work.
- Treat stop, wait, stand down, and pivot as control events for long-lived work.
- Match evidence to the claim. Use the smallest owning product-path check;
  add a falsifier for contested, load-bearing, or potentially vacuous claims.
  Record relevant controls, recovery, and blind spots without repeating proof.
- Evidence follows source and artifact identity, not the branch name. Reuse
  verified branch or merge-candidate evidence after landing when relevant code,
  build inputs, and dependencies are unchanged. Repeat affected checks only
  for a relevant change, observed failure, deployment, or packaging difference.
- "Ship" means integrated on owning main with terminal integration checks and
  applicable release or deployment checks complete. Confirm the landed change
  and merge result; do not rebuild or recapture screenshots solely for main.
- Ship a ready PR by adding the `ship` label when the repo has a Smart Ship
  caller (`.github/workflows/smart-ship.yml`); otherwise land through the merge
  queue with `gh pr merge --squash --auto`. Never `gh pr merge --admin`.
  Incidents use the org override labels `bypass-ci`, `bypass-merge-queue`, or
  `force-merge`.

<!-- END HARN SHARED AGENT CONTRACT -->
