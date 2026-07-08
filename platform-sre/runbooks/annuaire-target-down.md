# Runbook — AnnuaireTargetDown (severity: page)

## Signification
Prometheus ne scrape plus `annuaire` depuis **2 min** (`up == 0`). Soit les pods sont
down, soit le ServiceMonitor / le label `release: kps` est cassé.

## Diagnostic
1. Pods vivants ? `kubectl get pods -n devhub-dev -l app.kubernetes.io/name=annuaire`
2. Cible côté Prometheus : UI Prometheus → Status → Targets, chercher `annuaire`.
3. Le Service porte-t-il bien `release: kps` ? `kubectl get svc -n devhub-dev --show-labels`
   (piège n°1 du poly).
4. Endpoints peuplés ? `kubectl get endpoints -n devhub-dev`.

## Mitigation
- Pods down → voir logs / events (`kubectl describe pod`), corriger la cause (image, probe).
- Scrape cassé → vérifier `monitoring.enabled: true` et le label `release` dans `values`,
  commit, ArgoCD sync.

## Escalade
Si les pods crashent en boucle (CrashLoopBackOff) sans cause évidente → lead plateforme.

## Post-mortem
Distinguer « le service est tombé » de « on a juste perdu la visibilité ». Les deux sont graves.
