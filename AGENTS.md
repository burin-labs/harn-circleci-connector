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

- Build ambitious outcomes behind small typed interfaces; give behavior one owner
  and generate or parity-test projections instead of duplicating policy.
- Work autonomously within approved scope. Pause for destructive or production effects,
  exceptional spend, material ambiguity, or new authority.
- Treat stop, wait, stand down, pivot, and steer as control events.
- Use the smallest owning product-path check. Add a falsifier for contested, load-bearing,
  or potentially vacuous claims; record controls, recovery, and blind spots.
- Evidence follows source/artifact identity. Reuse proof when relevant code, build inputs,
  and dependencies are unchanged. Repeat affected checks for relevant changes, failures,
  deployment, or packaging differences. Do not rebuild or recapture solely for main.
- Ship means owning-main integration with terminal merge and applicable release/deploy
  checks. Confirm landed content and result; an open PR is incomplete.
- Use `ship` with a deployed Smart Ship caller; otherwise use `gh pr merge --squash --auto`.
  Never use `--admin`; incidents use `bypass-ci`, `bypass-merge-queue`, or `force-merge`.

<!-- END HARN SHARED AGENT CONTRACT -->
