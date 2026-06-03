# harn-circleci-connector

Pure-Harn CircleCI connector: signed webhooks (`workflow-completed` /
`job-completed`), CI-failure normalization, rerun-from-failed, and artifact
listing. Part of the Harn connectors program.

User story: receive a webhook when CI fails, map it to the PR / commit / branch,
and let an agent diagnose the failure or rerun the workflow.

## Install

```sh
harn add harn-circleci-connector
```

Set up secrets per binding:

| Secret                    | Used for                                                  |
| ------------------------- | -------------------------------------------------------- |
| `circleci/webhook-secret` | Verifies the `circleci-signature` HMAC on inbound hooks  |
| `circleci/api-token`      | Outbound REST API v2 as the `Circle-Token` header        |

## Inbound webhooks

CircleCI signs the **raw request body** with HMAC-SHA256 and ships the digest in
a comma-separated, versioned `circleci-signature` header (`v1=<hex>`). The
connector recomputes the highest known version, compares in constant time, and
**fails closed** when no signing secret is configured. There is no timestamp in
the signature (no replay window), so events are deduplicated on the top-level
delivery `id`, which CircleCI reuses on retries.

`normalize_inbound` returns a tagged envelope (`{type: "event" | "reject", ...}`)
whose event payload is a normalized CI shape:

```text
{
  provider: "circleci",
  kind: "workflow-completed" | "job-completed",
  status, is_failure,            // failure = status in {failed, error}
  repo, commit_sha, branch,      // from pipeline.vcs
  project_slug,                  // vcs/org/repo
  workflow_id, pipeline_number, web_url,
  job?,                          // present for job-completed
  rerun_handle: { workflow_id },
  triage: { ... },               // compact shape for failure triggers
  raw,                           // original payload
}
```

CircleCI payloads carry no reliable PR field. Resolve PRs out-of-band by SHA:

```harn
let pulls = circleci.github_commit_pulls("acme", "widgets", event.payload.commit_sha, github_token)
```

### Trigger recipe — rerun failed workflows from the last failure

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

## Outbound methods

Base URL `https://circleci.com/api/v2`, auth header `Circle-Token`. Mutating
methods (`workflow.rerun`, `workflow.cancel`) are flagged `requires_approval` by
`methods()`.

| Method            | Request                                                        |
| ----------------- | -------------------------------------------------------------- |
| `workflow.rerun`  | POST `/workflow/{id}/rerun` (`{from_failed: true}` or `{jobs}`) |
| `workflow.cancel` | POST `/workflow/{id}/cancel`                                    |
| `workflow.get`    | GET `/workflow/{id}`                                            |
| `workflow.jobs`   | GET `/workflow/{id}/job`                                        |
| `job.get`         | GET `/project/{project-slug}/job/{number}`                     |
| `artifacts.list`  | GET `/project/{project-slug}/{job-number}/artifacts`           |
| `api.request`     | Raw escape hatch: `{method, path, body?}`                       |

Outbound requests are https-only and reject loopback / private / link-local
hosts. Set `api_base_url` per call (or `CIRCLECI_API_BASE_URL`) for self-managed
hosts; pass `api_token` per call or set `CIRCLECI_API_TOKEN` / `CIRCLE_TOKEN`.

## Development

See [AGENTS.md](AGENTS.md) and the canonical
[connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md).
