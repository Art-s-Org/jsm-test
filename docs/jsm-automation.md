# The JSM side

Built in **Space settings → Automation → Flows** in your service space.

Nothing here touches the Jira REST API. GitHub only ever talks to a JSM incoming
webhook, and JSM only ever talks to GitHub's `dispatches` endpoint.

| | |
|---|---|
| Site | `https://<your-site>.atlassian.net` (stored as the `JSM_SITE_URL` variable, never committed) |
| Project | `EISM` |
| Repository | `Art-s-Org/jsm-test` |

---

## Step 1 — Three custom fields on the change request

This is what makes the control real. Without them, an approval can only mean
"deploy whatever is on main right now", which is not something you can defend in
a change review — main moves between raising the change and approving it.

Create these on the Change work item type:

| Field | Type | Holds |
|---|---|---|
| `Deployment repo` | Short text | `Art-s-Org/jsm-test` |
| `Deployment SHA` | Short text | the exact commit being deployed |
| `Deployment environment` | Dropdown (`staging`, `production`) | where it goes |

Two ways they get filled, and they converge on the same place:

- **Automatically** — Flow A below, if the incoming-webhook trigger is available
  in your Flows editor.
- **By hand** — the requester pastes the commit into the change request when
  they raise it. Perfectly fine for a pilot, and arguably more honest: the
  requester is declaring what they intend to ship.

Either way, **Flow B only reads the fields.** It does not care how they got there.

> Grab the field IDs while you are here: *Settings → Issues → Custom fields →*
> the field → the URL ends in `customfield_10xxx`. Smart values by name
> (`{{issue.Deployment SHA}}`) usually work, but the ID
> (`{{issue.customfield_10123}}`) always works. If Flow B sends an empty `sha`,
> the name lookup is what failed — switch to the ID.

---

## Step 2 — Flow B, "Release deployment on approval"

Build this one first. It is the actual gate, and it works whether or not Flow A
ever exists.

**Trigger:** `Approval completed`
*(the JSM-only trigger — "Flow is run when an approval is accepted or declined")*

**Condition 1** — Smart values condition:

```
{{approval.finalDecision}}  equals  approved
```

**Condition 2** — Work item fields condition:

```
Deployment SHA  is not empty
```

Condition 2 is the "a deployment must be attached to this change" rule. An
approved change with no commit recorded on it releases nothing.

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
    "sha": "{{issue.Deployment SHA}}",
    "environment": "{{issue.Deployment environment.value}}",
    "approved_by": "{{initiator.displayName}}"
  }
}
```

GitHub answers `204 No Content` — that is success, not an error. Leave
"Delay execution of subsequent rule actions until we've received a response"
unchecked.

### What GitHub does with that commit

`2 · Deploy` does not trust the payload. Before checking anything out it:

1. fails if `sha` is empty — nothing was attached to the change;
2. fails if the commit does not exist in the repository;
3. fails if the commit is not merged into the default branch.

Only then does it check out that exact commit. So the thing that deploys is the
thing the change request named, not whatever happened to be on main when the
approval landed.

---

## Step 3 — Flow A, "Record deployment on change request" *(optional)*

Only worth building if your Flows editor offers an **Incoming webhook** trigger.
It saves the requester from typing the commit by hand. Skip it otherwise —
nothing else depends on it.

**Trigger:** `Incoming webhook` → scope: **Work items provided in the request**

GitHub posts `{"issues": ["EISM-42"], "data": {...}}`. An unknown key matches no
work item and the flow silently does nothing, which is the behaviour we want.

Copy the generated URL into GitHub:

```
gh secret set JSM_WEBHOOK_URL --repo Art-s-Org/jsm-test
```

**If** `{{webhookData.data.event}}` equals `deployment-requested`:

- **Edit work item** → set `Deployment repo` = `{{webhookData.data.repo}}`,
  `Deployment SHA` = `{{webhookData.data.sha}}`,
  `Deployment environment` = `{{webhookData.data.environment}}`
- **Add comment**:

```
🚀 Deployment attached and waiting on this change.

Repository:  {{webhookData.data.repo}}
Commit:      {{webhookData.data.short_sha}}
Environment: {{webhookData.data.environment}}
Requested:   {{webhookData.data.requested_by}}
Pipeline:    {{webhookData.data.run_url}}

Nothing is live yet. Approving this change releases exactly this commit.
```

**Else if** `{{webhookData.data.event}}` equals `deployment-completed`:

- **Add comment**:

```
✅ Deployed to {{webhookData.data.environment}}.

Commit: {{webhookData.data.short_sha}}
URL:    {{webhookData.data.url}}
Run:    {{webhookData.data.run_url}}
```

- **Transition work item** → `Completed` (match the real status name in your
  change workflow; if it does not match, the action just fails harmlessly)

---

## Things that will bite you

- **`repository_dispatch` only runs workflows that exist on the default branch.**
  `deploy.yml` is already on `main`, so this is fine — just remember it if you
  start editing the workflow on a branch.
- **The PAT sits in plaintext in the rule.** JSM automation has no secret vault;
  anyone who can edit the flow can read the header. Use a *fine-grained* PAT
  scoped to this one repository with `Contents: Read and write` (the permission
  `dispatches` requires), and restrict who can edit flows.
- **Approval must actually be configured** on the EISM change workflow, or the
  `Approval completed` trigger never fires.
- **A declined approval also fires the trigger.** That is what Condition 1 is
  for — do not remove it.
- Use the **Audit log** tab next to Flows when a rule looks like it did nothing.
  It shows the payload received and which condition stopped it.
