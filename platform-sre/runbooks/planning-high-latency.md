# Runbook — PlanningHighLatencyP95 (severity: ticket)

## Signification
Latence p95 de `planning` > **400 ms** (SLO) depuis 30 min. Dégradation lente.

## Diagnostic
1. Dashboard RED `$app=planning` : p95 vs p50, corrélation release / charge.
2. `kubectl top pods -n devhub-dev -l app.kubernetes.io/name=planning`.

## Mitigation
- Release récente → `kubectl argo rollouts undo planning-dev-planning -n devhub-dev`.
- Charge → ajuster `replicaCount` / `resources`, commit, sync ArgoCD.

## Escalade
> 24 h au-dessus du SLO → revue de capacité avec l'équipe planning.

## Post-mortem
SLO 400 ms tenable ? Sinon réviser SLO ou dimensionnement.
