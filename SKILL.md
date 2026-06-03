# SKILL: harn-circleci-connector

Trigger recipes and outbound helpers for CircleCI via the pure-Harn
`harn-circleci-connector` package. CI-failure analog to the SCM connectors
(`harn-github-connector`, `harn-gitlab-connector`).

User story: receive a webhook on CI failure, map it to the PR / commit / branch,
then diagnose or rerun from an agent.

## What you get

- **Webhook inbound** with HMAC-SHA256 verification of the versioned
  `circleci-signature` header over the raw body (highest-version-only, fail
  closed, delivery-`id` dedup).
- **Normalized CI envelope** for `workflow-completed` and `job-completed`:
  `provider`, `kind`, `status`, `is_failure`, `repo`, `commit_sha`, `branch`,
  `project_slug`, `workflow_id`, `pipeline_number`, `web_url`, optional `job`,
  `rerun_handle`, and a `triage` shape.
- **REST API v2 outbound** for rerun-from-failed, cancel, workflow/job/artifact
  reads, and a raw `api.request` escape hatch (`Circle-Token` auth).
- **PR-by-SHA resolution** via the optional `github_commit_pulls` helper, since
  CircleCI payloads carry no PR field.

## Trigger recipe — rerun failed workflows from the last failure

```harn
import circleci from "harn-circleci-connector"

trigger ci_failed on circleci {
  source = {
    kind: "webhook",
    events: ["workflow-completed"],
  }
  on event {
    if event.payload.is_failure {
      circleci.call("workflow.rerun", {
        workflow_id: event.payload.rerun_handle.workflow_id,
        from_failed: true,
      })
    }
  }
}
```

## Trigger recipe — diagnose a failed job's artifacts

```harn
trigger diagnose_failure on circleci {
  source = {
    kind: "webhook",
    events: ["job-completed"],
  }
  on event {
    if event.payload.is_failure {
      let artifacts = circleci.call("artifacts.list", {
        project_slug: event.payload.project_slug,
        job_number: event.payload.job.job_number,
      })
      // Hand the artifacts + commit_sha to a diagnosing agent.
    }
  }
}
```

## Resolving the PR from a commit SHA

CircleCI does not include a PR number. Resolve it out-of-band:

```harn
let pulls = circleci.github_commit_pulls(
  "acme",
  "widgets",
  event.payload.commit_sha,
  env("GITHUB_TOKEN"),
)
```

## Required secrets per binding

| Secret                              | Used for                                              |
| ----------------------------------- | ----------------------------------------------------- |
| `circleci/webhook-secret`           | HMAC key for the `circleci-signature` header          |
| `circleci/api-token` / `api_token`  | Outbound REST v2 as the `Circle-Token` header         |

The signing secret is required for inbound webhooks — unsigned or mismatched
requests are rejected (fail closed). Configure it in the CircleCI webhook UI.

## Self-managed / custom hosts

Set `api_base_url` per call (or `CIRCLECI_API_BASE_URL`):

```harn
circleci.call("workflow.get", {
  api_base_url: "https://circleci.example.com/api/v2",
  workflow_id: "wf-...",
})
```

Pass `api_token` per call or set `CIRCLECI_API_TOKEN` / `CIRCLE_TOKEN`.
