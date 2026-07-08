# Runbook — AnnuaireHighLatencyP95 (severity: ticket)

## Signification
La latence p95 d'`annuaire` dépasse **300 ms** (SLO) depuis **30 min**. Dégradation
lente, pas une panne : **ticket**, pas page.

## Diagnostic
1. Dashboard RED `$app=annuaire`, panneau Latence : p95 vs p50 (saturation CPU ? GC ?).
2. CPU/mémoire des pods : `kubectl top pods -n devhub-dev -l app.kubernetes.io/name=annuaire`.
3. Corrélation avec un déploiement récent (panneau Build info) ou une montée de charge (RPS).

## Mitigation
- Si corrélé à une release : `git revert` ou `kubectl argo rollouts undo`.
- Si charge : augmenter `replicaCount` / `resources` dans `values.yaml`, commit, ArgoCD sync.

## Escalade
Si la latence reste > SLO 24 h → revoir le dimensionnement avec l'équipe annuaire.

## Post-mortem
Le SLO 300 ms est-il réaliste sous cette charge ? Ajuster SLO ou ressources.
