# Runbook — AnnuaireHighErrorRate (severity: page)

## Signification
Le taux de réponses 5xx d'`annuaire` dépasse **1 %** des requêtes depuis **5 min**.
Le service renvoie des erreurs serveur à une fraction non négligeable des utilisateurs.

## Impact
L'annuaire est utilisé par les équipes pédagogiques en continu. 1 % de 5xx soutenu
= consommation rapide de l'error budget (3 h 36/mois pour le SLO 99,5 %). **Page** justifié.

## Diagnostic (dans l'ordre)
1. Dashboard Grafana « DevHub Campus — RED par service », `$app=annuaire` : le pic est-il
   sur **toutes** les routes ou une seule ? sur la version **canary** ou **stable** ?
2. Un canary est-il en cours ?
   ```
   kubectl argo rollouts get rollout annuaire-dev-annuaire -n devhub-dev
   ```
3. Quelle route casse ?
   ```promql
   sum(rate(http_requests_total{app="annuaire",status_class="5xx"}[5m])) by (route)
   ```
4. Logs du pod fautif :
   ```
   kubectl logs -n devhub-dev -l app.kubernetes.io/name=annuaire --tail=100
   ```

## Mitigation
- **Si l'erreur vient d'un canary** : il devrait déjà être en rollback auto (AnalysisTemplate).
  Sinon, forcer :
  ```
  kubectl argo rollouts abort annuaire-dev-annuaire -n devhub-dev
  ```
  puis aligner Git : `git revert <commit du tag d'image>`.
- **Si l'erreur vient du stable** (pas de canary) : `git revert` du dernier commit de config,
  laisser ArgoCD resynchroniser.

## Escalade
Si le revert ne résout pas en 15 min → prévenir le lead plateforme + l'équipe annuaire.

## Post-mortem
Pourquoi l'AnalysisTemplate n'a pas attrapé la régression ? (seuil trop laxe ? route non couverte ?)
