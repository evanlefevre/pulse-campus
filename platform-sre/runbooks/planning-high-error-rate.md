# Runbook — PlanningHighErrorRate (severity: page)

## Signification
`planning` (le service **le plus critique**, SLO 99,9 %) dépasse **1 %** de 5xx depuis 5 min.
C'est le service dont une panne a déjà décalé 1200 emplois du temps : traiter en priorité.

## Diagnostic
1. Dashboard RED `$app=planning` : version active vs preview (blue/green).
2. `kubectl argo rollouts get rollout planning-dev-planning -n devhub-dev`
3. Par route : `sum(rate(http_requests_total{app="planning",status_class="5xx"}[5m])) by (route)`
4. Logs : `kubectl logs -n devhub-dev -l app.kubernetes.io/name=planning --tail=100`

## Mitigation
- Blue/green : rollback **instantané** en rebasculant l'activeService vers l'ancienne version
  (encore présente pendant `scaleDownDelaySeconds`) :
  ```
  kubectl argo rollouts undo planning-dev-planning -n devhub-dev
  ```
  puis aligner Git (`git revert`).

## Escalade
Erreur persistante après rollback → lead plateforme + équipe planning, immédiatement.

## Post-mortem
Le prePromotionAnalysis aurait-il dû bloquer la bascule ? Renforcer le seuil / la durée.
