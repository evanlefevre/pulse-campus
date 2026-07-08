# Runbook — PlanningTargetDown (severity: page)

## Signification
Prometheus ne scrape plus `planning` depuis 2 min (`up == 0`).

## Diagnostic
1. `kubectl get pods -n devhub-dev -l app.kubernetes.io/name=planning`
2. UI Prometheus → Status → Targets → `planning`.
3. Label `release: kps` sur le Service ? Endpoints peuplés ?

## Mitigation
- Pods down → logs/events, corriger (image, probe), sync.
- Scrape cassé → `monitoring.enabled: true` + label `release`, commit, sync.

## Escalade
CrashLoopBackOff inexpliqué → lead plateforme.

## Post-mortem
Perte de visibilité sur le service le plus critique = incident à part entière.
