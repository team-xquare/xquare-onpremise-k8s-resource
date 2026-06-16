# ci-system — Centralized CI eventing (SHADOW)

Runs **one** central EventBus + **one** receive-only GitHub EventSource for the whole
platform, replacing the current per-app footprint:

| | now (per-app) | this (central) |
|---|---|---|
| EventSource | 89 | **1** (receive-only) |
| Sensor | 89 | per-project (shadow: 2 echo) |
| EventBus (STAN) | 35 × 3 = 105 pods | **1 × 3** |
| GitHub webhooks | 86 auto-created | **0** created by Argo (App/org webhook feeds it) |
| installationId wiring | 29 | **0** |

It runs **in parallel** with existing CI and **does not touch it**. The shadow Sensors only
log a routing decision (echo Workflow) — no real build/deploy.

The k8s-resource ApplicationSet auto-creates an ArgoCD app `ci-system` (ns `ci-system`,
`CreateNamespace=true`, prune+selfHeal) the moment `charts/ci-system/` is pushed. **Nothing
happens until you commit & push.**

---

## How it works (verified against argo-events v1.9.3 source)

- EventSource has **no `apiToken` / `githubApp`** ⇒ `NeedToCreateHooks()=false` ⇒ Argo
  creates **no** webhook. It only receives POSTs.
- Every request is HMAC-verified: `parseValidateRequest → gh.ValidatePayload(r, secret)`
  against `webhookSecret`. Bad/absent signature ⇒ `request is not valid ... discarding`.
- `organizations: [team-xquare]` is only there to pass validation (one of
  repositories/organizations is required); it does **not** scope delivery in receive-only
  mode. Delivery scope = wherever the webhook is installed.
- The GitHub push body already contains `X-GitHub-Event` (Argo injects it), so the Sensor
  filters on `body['X-GitHub-Event']`, `body.repository.full_name`, `body.ref` — same shape
  your existing prod sensors already use.

---

## 1) Webhook secret (manual — not committed)

The chart references secret `github-central-webhook-secret` but does **not** create it
(keeps the value out of git). Create it once:

```bash
export KUBECONFIG=/path/to/kubeconfig
SECRET=$(openssl rand -hex 32)
kubectl create namespace ci-system --dry-run=client -o yaml | kubectl apply -f -
kubectl -n ci-system create secret generic github-central-webhook-secret \
  --from-literal=secret="$SECRET"
echo "use this same value as the GitHub webhook secret: $SECRET"
```

(For production, replace with a Vault `VaultStaticSecret` like the rest of the platform.)

## 2) Gateway host + DNS

- Point DNS `argo-events-shadow.dsmhs.kr` at the external gateway (same target as
  `argo-events-xquare-infra.dsmhs.kr`).
- Confirm the `external-gateway` listener `allowedRoutes` permits namespace `ci-system`
  (your per-project routes already attach cross-ns, so this is usually `From: All`).

## 3) Push & let ArgoCD sync
```bash
git add charts/ci-system && git commit -m "add ci-system shadow eventing" && git push
```
Watch the app come up:
```bash
kubectl -n ci-system get eventbus,eventsource,sensor,pods
```

## 4) Feed it (non-destructive)

Existing per-app webhooks are untouched. Add a **second** webhook on each repo you want to
observe (Settings → Webhooks → Add), e.g. `team-mozu/mozu-BE-v2`:

- Payload URL: `https://argo-events-shadow.dsmhs.kr/github`
- Content type: `application/json`
- Secret: the value from step 1
- Events: Just the push event (and the initial `ping` is sent automatically)

(Multi-org full rollout later replaces this with the GitHub App's single App-level webhook.)

---

## 5) Observe — parity checklist

```bash
# A. receipt + HMAC verification
kubectl -n ci-system logs deploy/github-central-eventsource -f
#   ✔ ping/push logged   ✖ "discarding" = secret mismatch

# B. filter + routing decision
kubectl -n ci-system logs -l app.kubernetes.io/component=shadow-sensor -f

# C. echo workflow = the routing decision, no real build
kubectl -n ci-system get wf
kubectl -n ci-system logs <shadow-echo-pod>   # "[SHADOW] would trigger CI: app=... sha=..."
```

Confirm before cutover:
1. Every push that triggered the **old** CI also reaches the central EventSource.
2. Wrong-signature POST is rejected (flip the secret to test).
3. Echo routing (app/branch) matches what the old per-app Sensor did.
4. Resource delta: `kubectl -n ci-system top pod` vs the ~290 per-app pods.

---

## 6) Cutover (phase 2 — separate change)

Once parity holds:
1. Move per-project **Sensors** into the project chart (rendered with `namespace: ci-system`),
   replacing the echo Workflow with the real `workflowTemplateRef` and
   `metadata.namespace: <project>-dsm-project` (dispatch into the project ns).
2. Add cross-ns RBAC: turn `ci-shadow-role` into a `ClusterRole` + a per-project
   `RoleBinding` so the central Sensor SA can create workflows in each project ns; the
   workflows still run as that project's `ci-workflow-sa`.
3. Repoint the GitHub **App-level** webhook (or per-org webhooks) to
   `https://<prod host>/github` and retire the per-app auto-created webhooks
   (delete the per-app EventSources, or set `deleteHookOnFinish`).

## Teardown (shadow)
Delete the repos' extra webhook, remove `charts/ci-system/`, push. ArgoCD prunes the app.
Existing CI is unaffected throughout.
