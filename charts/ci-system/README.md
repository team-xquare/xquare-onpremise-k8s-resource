# ci-system — Centralized CI eventing

One **EventBus** + one **receive-only GitHub EventSource** shared by every project, fed by
the **XQUARE-INFRASTRUCTURE GitHub App's single webhook**. Per-project/app **Sensors live in
the project gitops-repo chart** (`ci-central-sensor.yaml`) and route off this one stream —
nothing project-specific lives here.

Replaces the old per-app footprint:

| | per-app (old) | central (this) |
|---|---|---|
| EventSource | 89 | **1** (receive-only) |
| Sensor | 89 | per-project, in this ns (gitops-repo chart) |
| EventBus (STAN) | 35 × 3 = 105 pods | **1 × 3** |
| GitHub webhooks | 86 auto-created | **0** created by Argo (App webhook feeds it) |
| installationId wiring | 29 | **0** |

The k8s-resource ApplicationSet auto-creates an ArgoCD app `ci-system` (ns `ci-system`).

## How it works (argo-events v1.9.3)

- No `apiToken` / `githubApp` on the EventSource ⇒ `NeedToCreateHooks()=false` ⇒ Argo
  creates **no** webhook. It only receives POSTs.
- Every request is HMAC-verified (`gh.ValidatePayload` against `webhookSecret`); bad/absent
  signature is discarded.
- `organizations: [team-xquare]` only satisfies validation (receive-only doesn't scope on it;
  delivery scope = wherever the App is installed).
- GitHub injects `X-GitHub-Event` into the body, so project Sensors filter on
  `body['X-GitHub-Event']`, `body.repository.full_name`, `body.ref`.

## What this chart contains (central only)

`EventBus` · `EventSource` (receive-only) · `HTTPRoute` · `ServiceAccount ci-central-sensor-sa`
\+ `ClusterRole ci-central-sensor-wf` (cross-ns workflow create) · Vault `VaultAuth` +
`VaultStaticSecret`. **No Sensors.**

## Setup

**1) Webhook secret (Vault)** — same pattern as the project chart's `ci-github-secrets`:
```bash
vault kv put xquare-infra-kv/ci-central-webhook secret=$(openssl rand -hex 32)
```
Reuse that value as the GitHub App webhook secret (step 3).

**2) DNS + gateway** — point `webhook.host` (default `argo-events-ci.dsmhs.kr`) at the external
gateway; the listener `allowedRoutes` must permit ns `ci-system` (usually `From: All`).

**3) GitHub App (XQUARE-INFRASTRUCTURE)** — Webhook URL = `https://<webhook.host>/github`,
secret = step 1, content type `application/json`. Permissions: Contents `Read` (+ Metadata).
**Subscribe to events: Push** (each installation must accept the update before push events
flow). Installed on the orgs you want to build, with repo access.

## Opt a project into central eventing

In the project gitops-repo, set `centralEventing: true` in that project's values file. That
renders the per-project Sensors (in `ci-system`) + a RoleBinding for the central SA. To avoid
double builds, also disable that project's old per-app Sensor (guard `ci-sensor.yaml` /
`ci-event-source.yaml`, or scale it to 0) as you migrate.

## Verify

```bash
kubectl -n ci-system logs deploy/github-central-eventsource -f   # receipt + HMAC
kubectl -n ci-system get eventbus,eventsource                    # health
kubectl -n <project>-dsm-project get wf -w                       # dispatched builds
```
