# The JSM side

Two flows, built in **Space settings → Automation → Flows** in
*Enercare IT Service Management*. Nothing here touches the Jira REST API — the
GitHub side only ever talks to an incoming webhook, and JSM only ever talks to
GitHub's `dispatches` endpoint.

Values used below:

| | |
|---|---|
| Site | `https://<your-site>.atlassian.net` (stored as the `JSM_SITE_URL` variable, never committed) |
| Project | `EISM` |
| Repository | `Art-s-Org/jsm-test` (change if you use another repo) |

---

## Flow A — "Record deployment on change request"

Receives deployments from GitHub and pins them to the change request. Also
handles the write-back after a successful deploy.

**Trigger:** `Incoming webhook`

- Scope: **Work items provided in the request** — this is what makes GitHub's
  `{"issues": ["EISM-42"]}` body resolve to a real change request. An unknown or
  fake key matches nothing and the flow silently does nothing, which is exactly
  the behaviour we want.
- Copy the generated webhook URL. It goes into GitHub as the secret
  `JSM_WEBHOOK_URL`.

**Branch 1 — a deployment is being attached**

Condition (Smart values condition):

```
{{webhookData.data.event}}  equals  deployment-requested
```

Action 1 — *Add comment*:

```
🚀 Deployment attached and waiting on this change.

Repository:  {{webhookData.data.repo}}
Commit:      {{webhookData.data.short_sha}}
Environment: {{webhookData.data.environment}}
Requested:   {{webhookData.data.requested_by}}
Pipeline:    {{webhookData.data.run_url}}

Nothing is live yet. Approving this change releases the deployment automatically.
```

Action 2 — *Edit work item* → Labels → **Add** `deployment-attached`

> This label is the enforcement point. Flow B refuses to release anything
> without it, so an approved change with no deployment attached deploys nothing.

**Branch 2 — the deploy finished**

Condition:

```
{{webhookData.data.event}}  equals  deployment-completed
```

Action 1 — *Add comment*:

```
✅ Deployed to {{webhookData.data.environment}}.

Commit: {{webhookData.data.short_sha}}
URL:    {{webhookData.data.url}}
Run:    {{webhookData.data.run_url}}
```

Action 2 — *Transition work item* → `Completed`
(match this to whatever your change workflow actually calls the post-implementation
status; if the transition name differs the action just fails harmlessly.)

---

## Flow B — "Release deployment on approval"

This is the one you already found in the UI.

**Trigger:** `Approval completed`
*(the JSM-only trigger — "Flow is run when an approval is accepted or declined")*

**Condition 1** — Smart values condition:

```
{{approval.finalDecision}}  equals  approved
```

**Condition 2** — Work item fields condition:

```
Labels  contains  deployment-attached
```

Without this second condition an approval on a change that never had a
deployment attached would still fire GitHub. With it, approval releases *only*
what was actually attached.

**Action:** `Send web request`

| Field | Value |
|---|---|
| URL | `https://api.github.com/repos/Art-s-Org/jsm-test/dispatches` |
| Method | `POST` |
| Web request body | Custom data |

Headers:

```
Authorization        Bearer <GITHUB_PAT>
Accept               application/vnd.github+json
X-GitHub-Api-Version 2022-11-28
```

Body:

```json
{
  "event_type": "jsm-approved",
  "client_payload": {
    "issue_key": "{{issue.key}}",
    "environment": "production",
    "approved_by": "{{initiator.displayName}}"
  }
}
```

GitHub answers `204 No Content` — that is success, not an error. Leave
"Delay execution of subsequent rule actions until we've received a response"
unchecked.

---

## Things that will bite you

- **`repository_dispatch` only runs workflows that exist on the default branch.**
  Merge `deploy.yml` to `main` before testing, or the webhook returns 204 and
  nothing happens.
- **The PAT is stored in plaintext in the rule.** JSM automation has no secret
  vault; anyone who can edit the flow can read the header. Use a *fine-grained*
  PAT scoped to this one repository with `Contents: Read and write` (that is the
  permission `dispatches` requires), and restrict who can edit flows.
- **`environment` is hardcoded to `production`** in Flow B, because a smart value
  from Flow A's webhook isn't available later in Flow B. To make it dynamic, have
  Flow A write it to a custom field (e.g. *Deployment environment*) and read it
  back in Flow B as `{{issue.Deployment environment}}`.
- **Approval must actually be configured** on the EISM change workflow, otherwise
  the `Approval completed` trigger never fires.
- Use the **Audit log** tab next to Flows when a rule looks like it did nothing —
  it shows the payload it received and which condition stopped it.
