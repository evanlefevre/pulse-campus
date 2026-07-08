# Runbook — NotifHighLatencyP95 (severity: ticket)

## Signification
Latence p95 de `notif` > **500 ms** (SLO) depuis 30 min. Service non critique.

## Diagnostic
1. Dashboard RED `$app=notif` : p95 vs p50.
2. `kubectl top pods -n devhub-dev -l app.kubernetes.io/name=notif`.

## Mitigation
- Release récente → `kubectl rollout undo deployment/notif-dev-notif -n devhub-dev`.
- Charge → ajuster ressources/replicas, commit, sync.

## Escalade
Peu probable. Traiter en heures ouvrées.

## Post-mortem
Vérifier que le SLO 500 ms reste pertinent pour l'usage réel de notif.
