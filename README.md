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
- JSM Flow B only fires on an approved approval **and** only if a deployment was
  actually attached (the `deployment-attached` label).
- A made-up change key resolves to no work item in JSM, so Flow A does nothing,
  no label is set, and approval can never release anything.

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

1. Raise a change request in **EISM**. Note the key, e.g. `EISM-42`.
2. Actions → **1 · Request deploy** → run it with that key. It attaches and stops.
   Check the ticket: there is a comment and a `deployment-attached` label.
3. Run it again with a key that does not exist — the flow no-ops, no label, and
   approving that change later releases nothing.
4. Approve `EISM-42` in JSM. Within seconds **2 · Deploy** starts on its own,
   publishes the page, cuts release `deploy-production-N`, and comments the
   result back on the ticket.
