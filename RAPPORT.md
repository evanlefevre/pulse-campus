# Rapport TP3 — Livraison progressive et observabilité

**Auteur :** Évan Lefevre — ESGI 5 SRC — 2025/2026
**Dépôt :** https://github.com/evanlefevre/pulse-campus

But du TP : ajouter à la chaîne GitOps du TP2 de la livraison progressive (Argo Rollouts) et de
l'observabilité (Prometheus, Grafana, Alertmanager), pour ne promouvoir une nouvelle version que si
les métriques sont bonnes. Cluster local kind `pulse` (2 nœuds), 3 services : annuaire (canary),
planning (blue/green), notif (classique).

> Les lignes **📸 CAPTURE** indiquent une capture d'écran à insérer, et ce qu'elle doit montrer.

---

## Étape 0 — Outils

| Outil | Version |
|---|---|
| kubectl-argo-rollouts | 1.9.0 |
| promtool | 2.53.2 |
| jq | 1.8.2 |
| kubectl / helm / kind | 1.33 / 3.17 / 0.31 |

`make tools-check-tp3` vérifie la présence et la version minimale de chaque outil (sort en erreur si
un outil manque). Sortie : tous les outils présents et à jour.

---

## Étape 1 — SLI, SLO, error budget

- **SLI** : un signal mesuré (ex : part des requêtes sans erreur 5xx sur 5 min).
- **SLO** : un objectif chiffré sur un SLI, avec une fenêtre (ex : 99,5 % sur 30 jours).
- **Error budget** : la marge d'erreur tolérée = (1 − SLO) × fenêtre.

On pilote en **RED** (Rate, Errors, Duration), adapté aux services web. Fenêtre de 30 jours = 43 200 min.

| Service | SLO dispo | Error budget / mois | SLO latence p95 |
|---|---|---|---|
| annuaire | 99,5 % | 216 min | < 300 ms |
| planning | 99,9 % | 43 min | < 400 ms |
| notif | 99,0 % | 432 min | < 500 ms |

PromQL du SLI de disponibilité (annuaire) :
```promql
sum(rate(http_requests_total{app="annuaire", status_class!~"5.."}[5m]))
/ sum(rate(http_requests_total{app="annuaire"}[5m]))
```

Si l'error budget est épuisé trop vite, on gèle les nouvelles fonctionnalités et on ne déploie que
des correctifs jusqu'à ce qu'il se reconstitue.

---

## Étape 2 — Instrumentation Prometheus

Le code des services n'est pas modifié (interdit par le sujet). Chaque service expose déjà
`http_requests_total`, `http_request_duration_seconds` (histogramme) et `<svc>_build_info`. Les
buckets de l'histogramme sont réglés par la variable d'environnement `METRICS_BUCKETS`, alignés sur
le SLO de latence de chaque service. Pour annuaire (SLO 300 ms) : `0.05,0.1,0.2,0.3,0.5,1,2`.

Vérification sur l'image en local :
```
curl -s localhost:8080/metrics | grep http_request_duration_seconds_bucket
```
Les buckets exposés sont bien `le="0.05" … 0.3 … 0.5 … 2 … +Inf`, donc le bucket à 0.3 s (= le SLO)
est présent, ce qui permet un p95 précis au seuil.

---

## Étape 3 — kube-prometheus-stack via ArgoCD

Prometheus + Grafana + Alertmanager sont installés par une Application ArgoCD (chart Helm figé en
version `65.5.0`, values dans le repo). Le mot de passe Grafana est sorti de Git via un Secret
`grafana-admin` créé à la main. Sur kind, j'ai désactivé les sous-charts de la control-plane
(kubeControllerManager, kubeScheduler, kubeEtcd, kubeProxy) car ils n'ont pas de cible exposée.

Déploiement : `kubectl apply -f platform-sre/bootstrap/root-app.yaml` (seul apply manuel), puis
ArgoCD crée tout le reste.

> 📸 **CAPTURE 1** — UI ArgoCD (port-forward `svc/argocd-server` puis https://localhost:8080) :
> l'application **kube-prometheus-stack** en **Synced / Healthy**, et la liste des applications toutes
> vertes.

---

## Étape 4 — ServiceMonitor + dashboard Grafana

Chaque chart a un `ServiceMonitor` (label `release: kps` obligatoire, sinon Prometheus l'ignore) qui
sélectionne le Service générique du service. Un relabeling remonte `rollouts_pod_template_hash` (pour
distinguer canary et stable à l'étape 7) et un label `app`.

Le dashboard Grafana « DevHub Campus — RED par service » est un seul dashboard générique (variables
`namespace` et `app`) chargé en GitOps. Ses 4 panneaux : RPS par route, taux d'erreur 5xx, latence
p50/p95/p99, et build_info (version active).

> 📸 **CAPTURE 2** — Prometheus, menu **Status → Targets** : les 3 services `devhub-dev` en **UP**.
>
> 📸 **CAPTURE 3** — Grafana, dashboard **DevHub Campus — RED par service** (namespace=devhub-dev,
> app=annuaire) avec du trafic : les 4 panneaux qui affichent des courbes.

---

## Étape 5 — Passage du Deployment au Rollout (canary)

Le service annuaire passe de `Deployment` à `Rollout` (stratégie canary). Deux Services (stable et
canary), l'ingress pointe sur le stable, et Argo Rollouts gère le pourcentage de trafic via
l'ingress-nginx. Étapes du canary : 25 % → analyse → 50 % → 100 %.

Pour déclencher un déploiement sans rebuild, je change une variable dans les values (ex : `LOG_LEVEL`)
puis je commit/push. ArgoCD synchronise et le canary démarre. Suivi avec :
```
kubectl argo rollouts get rollout annuaire-dev-annuaire -n devhub-dev
```
Résultat pendant un canary : Status `Paused`, Step 1/6, SetWeight 25, un ReplicaSet canary (1 pod) et
un ReplicaSet stable (2 pods).

> 📸 **CAPTURE 4** — Dashboard Argo Rollouts (http://rollouts.devhub.local) : annuaire en **Canary** et
> planning en **BlueGreen**.

---

## Étape 6 — Canary manuel (promote / abort)

| Commande | Résultat |
|---|---|
| `kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev` | passe à l'étape suivante |
| `kubectl argo rollouts abort annuaire-dev-annuaire -n devhub-dev` | canary à 0, retour au stable (Degraded) |
| `kubectl argo rollouts promote ... --full` | promeut directement à 100 % |

`promote --full` est acceptable seulement en urgence (hotfix vérifié par ailleurs), car on saute
l'analyse. À noter : `abort` ramène le cluster à l'ancienne version mais ne modifie pas Git, il y a
donc un écart à corriger ensuite (git revert ou promote).

> 📸 **CAPTURE 5** — Dashboard Rollouts pendant un canary en cours (poids 25 %, revision canary vs
> revision stable, bouton Promote actif).

---

## Étape 7 — AnalysisTemplate : promotion sur métriques

Entre 25 % et 50 %, j'ajoute une étape d'analyse qui interroge Prometheus pendant 5 min (10 mesures
toutes les 30 s). Deux métriques sur la version canary uniquement (filtrée par
`rollouts_pod_template_hash`) : taux de 5xx < 1 % et latence p95 < 300 ms. Une mesure ratée entraîne
le rollback.

**Cas normal** (trafic sain) : l'analyse passe, le rollout monte tout seul à 50 % puis 100 %.
```
AnalysisRun : Successful
  error-rate  = 0 %     (< 1 %)
  latency-p95 = 47 ms   (< 300 ms)
```

**Cas dégradé** (`FAIL_RATE=0.5` sur la nouvelle version, trafic sur /break) : l'analyse échoue et le
rollout revient tout seul au stable.
```
AnalysisRun : Failed  ->  Rollout Degraded, SetWeight 0
  error-rate = 16 %  (> 1 %)
```
La mauvaise version n'a jamais dépassé 25 % du trafic et a été retirée en moins d'une minute.

> 📸 **CAPTURE 6** — l'AnalysisRun **Successful** (ou le rollout promu à 100 % après analyse).
>
> 📸 **CAPTURE 7** — l'AnalysisRun **Failed** et le rollout revenu au stable (rollback automatique).

---

## Étape 8 — Blue/green (planning)

Le service planning est en blue/green : la nouvelle version (preview) tourne en parallèle de l'active,
avec une analyse de pré-promotion de 2 min avant de basculer. `autoPromotionEnabled: false` (bascule
manuelle), et l'ancienne version est gardée 5 min après la bascule (rollback instantané possible). La
preview est exposée sur `planning-preview.devhub.local`.

```
kubectl get pods -n devhub-dev -l app.kubernetes.io/name=planning   # 4 pods pendant la bascule
kubectl argo rollouts promote planning-dev-planning -n devhub-dev   # bascule active -> nouvelle
```

Différence avec le canary : les deux versions tournent à pleine capacité (2× les ressources), la
bascule est totale d'un coup, et le rollback est immédiat.

> 📸 **CAPTURE 8** — Dashboard Rollouts sur planning : les deux ReplicaSets (active + preview) pendant
> la bascule, ou les deux `curl` (preview vs active).

---

## Étape 9 — Routage par header

Pour tester une version en interne avant de l'ouvrir au public, le rollout canary ajoute une règle sur
l'ingress : les requêtes avec le header `X-Beta-User: true` vont sur le canary, même si son poids est à
0. J'ai vérifié les annotations de l'ingress canary généré :
```
nginx.ingress.kubernetes.io/canary-by-header: "X-Beta-User"
nginx.ingress.kubernetes.io/canary-weight: "0"
```
Donc le trafic normal reste 100 % sur le stable, et seul le header force le canary.

---

## Étape 10 — Alertmanager + notifications Rollouts

Chaque service a des règles d'alerte (`PrometheusRule`), validées par `promtool` (annuaire : 3 règles,
planning : 3, notif : 2). Gradation :
- HighErrorRate → **page** (réveille, `for: 5m`)
- HighLatencyP95 → **ticket** (`for: 30m`)
- TargetDown → **page**

Alertmanager route page et ticket vers deux webhooks différents (mockés sur webhook.site), avec une
règle d'inhibition (si une page part, pas de ticket en double). Les Rollouts envoient aussi une
notification en cas de promotion ou d'abort.

Tests réels :
- Rollout promu / aborté → webhook reçu avec `conclusion: Promoted` puis `Aborted`.
- FAIL_RATE=0.8 + trafic /break → l'alerte `AnnuaireHighErrorRate` passe en firing après 5 min, et le
  webhook page reçoit l'alerte (severity page, summary, runbook_url).

> 📸 **CAPTURE 9** — webhook.site (rollout) : la requête reçue avec le JSON `conclusion: Promoted`.
>
> 📸 **CAPTURE 10** — webhook.site (page) : l'alerte `AnnuaireHighErrorRate`, severity page.

---

## Étape 11 — Comparaison RollingUpdate / Argo Rollouts / Flagger

| Critère | RollingUpdate | Argo Rollouts | Flagger |
|---|---|---|---|
| Simplicité | 5 (natif K8s) | 3 | 2 |
| Stratégies (canary, blue/green, analyse) | 1 | 5 | 4 |
| Analyse par métriques | 0 | 5 | 4 |
| UI / dashboard | 1 | 5 | 2 |
| Intégration ArgoCD | 4 | 5 | 3 |
| Coût (composant en plus) | 5 | 3 | 3 |

RollingUpdate gagne sur la simplicité, mais ne sait pas mesurer. Argo Rollouts apporte les stratégies
+ l'analyse + une UI, et s'intègre bien avec ArgoCD (même éditeur), ce qui colle à la chaîne du TP2.
Flagger est l'équivalent côté Flux / service mesh.

---

## Étape 12 — Synthèse

Ce que le TP3 apporte par rapport au TP2 : on ne déploie plus « à l'aveugle ». Avant, ArgoCD disait
seulement que le cluster était conforme à Git, pas que le service marchait. Maintenant, une nouvelle
version n'atteint 100 % du trafic que si Prometheus confirme qu'elle ne dégrade ni les erreurs ni la
latence. Le vrai gain, c'est de limiter l'impact d'une mauvaise version (25 % du trafic pendant moins
d'une minute au lieu de toute la flotte).

Ce que la chaîne ne fait toujours pas, et ce que j'ajouterais ensuite :
- **Tracing distribué** (OpenTelemetry + Jaeger) pour savoir où naît une latence entre services.
- **Logs centralisés** (Loki) corrélés aux métriques.
- **Policy d'admission** (Kyverno) pour empêcher un déploiement sans AnalysisTemplate.
- **Secrets** propres (Sealed Secrets / External Secrets) au lieu du mot de passe Grafana créé à la main.

---

## Problèmes rencontrés (et corrigés)

Le squelette fourni ne se déployait pas tel quel. J'ai dû corriger :
- l'AppProject qui n'autorisait pas les namespaces `argocd` et `kube-system` ;
- l'operator Prometheus bloqué (désactivé `prometheusOperator.tls`) ;
- le port des Rollouts sans `protocol: TCP` (erreur de diff ServerSideApply) ;
- les ServiceMonitor sans `action: replace` (OutOfSync permanent) ;
- la requête error-rate des AnalysisTemplate qui plantait quand il n'y avait aucune erreur (ajout de
  `or vector(0)`).
