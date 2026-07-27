# JSM change-gated deployments — pilot

A deployment cannot happen unless it is attached to an **existing** change
request in Jira Service Management, and that change request is **approved**.

Nothing real is deployed. The artifact is a single `index.html` published to
GitHub Pages, so the whole thing still shows up under **Deployments**,
**Environments** and **Releases** in the GitHub UI.

## How it works

```
     ┌──────────────────────────────── GitHub ────────────────────────────────┐
     │                                                                        │
 you ──▶ 1 · Request deploy ──── webhook ────▶ JSM Flow A                      │
     │   (workflow_dispatch,                  records the deployment on        │
     │    needs a change key)                 EISM-42, labels it              │
     │        ⏹ stops here                    "deployment-attached"           │
     │                                                                        │
     │                                        ⏸ someone approves the change   │
     │                                                                        │
     │   2 · Deploy  ◀──── repository_dispatch ──── JSM Flow B                 │
     │   publishes the page,                  "Approval completed"            │
     │   cuts a release,                      + finalDecision = approved      │
     │   posts back to JSM ───── webhook ───▶ + label present                 │
     └────────────────────────────────────────────────────────────────────────┘
```

The gate is structural, not advisory:

- `2 · Deploy` has **no** `push` and **no** `workflow_dispatch` trigger. Its only
  trigger is `repository_dispatch: [jsm-approved]`, which only JSM sends.
- JSM Flow B only fires on an **approved** approval, and only if the change
  request actually carries a `Deployment SHA`.
- GitHub then re-checks the payload before touching anything: the commit must
  exist in this repository and must already be merged to the default branch.
  **The commit that deploys is the one the change request named** — not whatever
  happened to be on `main` when the approval landed.

## Files

| | |
|---|---|
| [.github/workflows/request-deploy.yml](.github/workflows/request-deploy.yml) | Attaches a deployment to a change request. Deploys nothing. |
| [.github/workflows/deploy.yml](.github/workflows/deploy.yml) | Runs only when JSM says the change was approved. |
| [site/index.html](site/index.html) | The "application". Stamped with the change key at deploy time. |
| [docs/jsm-automation.md](docs/jsm-automation.md) | The two JSM flows, click by click. |

## Setup

1. **Enable Pages** — Settings → Pages → Source: **GitHub Actions**.
2. **Build the JSM flows** — follow [docs/jsm-automation.md](docs/jsm-automation.md).
   Flow A gives you a webhook URL; Flow B needs a GitHub PAT.
3. **GitHub secrets** (Settings → Secrets and variables → Actions):
   - `JSM_WEBHOOK_URL` — the incoming webhook URL from Flow A.
   - Variable `JSM_SITE_URL` — `https://<your-site>.atlassian.net`
4. **Merge to the default branch.** `repository_dispatch` only triggers workflows
   that exist on the default branch.

Optionally add required reviewers to the `production` environment
(Settings → Environments) for a second, GitHub-side gate.

## Try it

1. Raise a change request in **EISM**. Set `Deployment SHA` to a commit on `main`
   and `Deployment environment` to `production`. Note the key, e.g. `EISM-42`.
2. Approve it. Within seconds **2 · Deploy** starts on its own, checks out that
   exact commit, publishes the page, cuts release `deploy-production-N`, and
   comments the result back on the ticket.

Then prove the gate actually holds:

3. Approve a change with `Deployment SHA` empty → Flow B's condition stops it,
   nothing is dispatched, nothing deploys.
4. Put a commit that only exists on a branch into `Deployment SHA` and approve →
   GitHub dispatches, then fails closed with "not merged into main".
5. Decline an approval → the trigger fires, but `finalDecision` is not
   `approved`, so nothing leaves JSM.
