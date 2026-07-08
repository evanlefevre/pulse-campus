# Runbook — NotifHighErrorRate (severity: ticket)

## Signification
`notif` dépasse **2 %** de 5xx depuis 10 min. Service **non critique** (SLO 99 %, reste un
Deployment classique) : pas de page, un **ticket** suffit.

## Diagnostic
1. Dashboard RED `$app=notif`, par route.
2. `kubectl logs -n devhub-dev -l app.kubernetes.io/name=notif --tail=100`
3. `kubectl rollout status deployment/notif-dev-notif -n devhub-dev`

## Mitigation
notif est en RollingUpdate natif (pas de canary) : rollback via
`kubectl rollout undo deployment/notif-dev-notif -n devhub-dev`, puis `git revert`.

## Escalade
Rare ; si les notifications bloquent un workflow critique, remonter à l'équipe notif.

## Post-mortem
notif mériterait-il de passer en Rollout canary comme annuaire ? (cf. synthèse étape 12)
