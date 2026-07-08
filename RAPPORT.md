# RAPPORT — TP 3 · DevHub Campus SRE · Livraison progressive & observabilité

> **Binôme :** _Évan Lefevre_ + _<binôme>_ · ESGI 5 SRC · 2025-2026
> **Dépôt :** `https://github.com/evanlefevre/pulse-campus`
>
> ⚠️ **Avant de déployer :** remplacez partout `evanlefevre` par votre handle GitHub
> (manifestes `platform-sre/**`, `runbook_url`), et `ghcr.io/changeme/<svc>` par votre
> compte GHCR (`services/*/chart/values.yaml`).
>
> 🟦 Les blocs notés **`[À EXÉCUTER + CAPTURE]`** demandent un cluster vivant : exécutez la
> commande et collez la capture / sortie à l'endroit indiqué. Tout le reste (manifestes, PromQL,
> argumentaire) est déjà écrit et validé (`helm lint` + `helm template` OK sur les 3 charts).

---

## Fil rouge

> *« ArgoCD nous dit que l'état du cluster correspond au Git. Il ne nous dit pas que le service
> fonctionne. »*

Le TP 2 nous a fait **pousser plus vite** ; le TP 3 nous fait **pousser plus sûr**. On ne
déploie plus rien sans le mesurer, et on ne promeut rien qui dégrade les métriques. Concrètement,
on insère entre « ArgoCD synchronise » et « 100 % du trafic » une **boucle de preuve** :
Prometheus mesure → un `AnalysisTemplate` décide → Argo Rollouts promeut ou rollback.

---

## Étape 0 — Outillage TP 3

| Outil | Rôle | Version minimale | Version installée |
|---|---|---|---|
| `kubectl-argo-rollouts` | piloter les Rollouts (status, promote, abort, dashboard) | 1.7 | `[À COMPLÉTER]` |
| `promtool` | valider PrometheusRules hors cluster | 2.50 | `[À COMPLÉTER]` |
| `amtool` (optionnel) | tester les routes Alertmanager | 0.27 | `[À COMPLÉTER]` |
| `jq` | parser l'API Prometheus en JSON | 1.7 | `[À COMPLÉTER]` |

**`[À EXÉCUTER + CAPTURE]`** — coller la sortie de :
```bash
kubectl argo rollouts version
promtool --version
jq --version
make tools-check-tp3      # cible ajoutée au Makefile (défi bonus étape 0)
```

> **Défi bonus réalisé :** la cible `make tools-check-tp3` vérifie *présence ET version minimale*
> de chaque outil (TP 2 + TP 3) et sort en `exit 1` explicite si un outil manque ou est trop ancien.
> Voir [`Makefile`](Makefile).

**Pièges notés :** `kubectl-argo-rollouts` est un *plugin* (`kubectl argo rollouts …`), pas une CLI
standalone. Ne pas confondre `argocd`, `argo` (Workflows) et `kubectl argo rollouts`.

---

## Étape 1 — SLI, SLO, error budget

**Définitions retenues (livre Google SRE) :**
- **SLI** = un *signal mesurable* (ex. « part des requêtes non-5xx sur 5 min »). Pas une métrique brute.
- **SLO** = un *seuil* sur un SLI, avec **fenêtre** (ex. 99,5 % sur 30 j). Sans fenêtre, ça ne veut rien dire.
- **SLA** = engagement *contractuel* (avec pénalité). Interne ≠ contractuel.
- **Error budget** = la part du SLO qu'on peut « brûler » = `(1 − SLO) × fenêtre`.

**RED vs USE :** RED (Rate, Errors, Duration) = orienté **requêtes** (notre cas, services web).
USE (Utilization, Saturation, Errors) = orienté **ressources** (CPU, mémoire, disque). On pilote
les SLO en RED et on garde USE pour le diagnostic de capacité.

**Calcul error budget mensuel (fenêtre 30 j = 43 200 min) :** `budget = (1 − SLO) × 43 200`.

### annuaire (service de lecture, peu critique)
| SLI | SLO | Error budget mensuel |
|---|---|---|
| Disponibilité (succès non-5xx) | 99,5 % | (1−0,995)×43200 = **216 min ≈ 3 h 36** |
| Latence p95 (fenêtre 5 min) < 300 ms | 99 % du temps sous le seuil | 1 %×43200 = **432 min ≈ 7 h 12** au-dessus toléré |
| Fraîcheur de version (la flotte sert le tag attendu < 10 min après promotion) | 99 % | **216 min** de dérive tolérée |

### planning (cœur métier — l'incident d'origine, le plus critique)
| SLI | SLO | Error budget mensuel |
|---|---|---|
| Disponibilité | **99,9 %** | (1−0,999)×43200 = **43,2 min** |
| Latence p95 < 400 ms | 99,9 % | **43 min** au-dessus toléré |
| Fraîcheur de version < 5 min après bascule blue/green | 99,9 % | **43 min** de dérive tolérée |

### notif (notifications — dégradation tolérable)
| SLI | SLO | Error budget mensuel |
|---|---|---|
| Disponibilité | 99,0 % | (1−0,99)×43200 = **432 min ≈ 7 h 12** |
| Latence p95 < 500 ms | 99 % | **432 min** au-dessus toléré |
| Fraîcheur de version < 30 min | 99 % | **432 min** de dérive tolérée |

> **Choix « Fraîcheur de version » plutôt que « Fraîcheur des données » :** les 3 services sont
> stateless (CRUD en mémoire), la fraîcheur de *donnée* n'a pas de sens. En revanche, dans une
> chaîne de *progressive delivery*, un SLI très parlant est : *« 100 % de la flotte sert-elle la
> version attendue, et combien de temps après la promotion ? »*. C'est mesurable via
> `<svc>_build_info` et c'est exactement ce que le TP construit.

### PromQL pseudo-code des SLI
```promql
# Disponibilité (annuaire) — part des requêtes non-5xx
sum(rate(http_requests_total{app="annuaire", status_class!~"5.."}[5m]))
/
sum(rate(http_requests_total{app="annuaire"}[5m]))

# Latence p95 (annuaire)
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{app="annuaire"}[5m])) by (le)
)

# Fraîcheur de version (annuaire) — la flotte sert-elle le commit attendu ?
count(annuaire_build_info{commit="<tag-attendu>"}) / count(annuaire_build_info)
```

> **Validation orale (à restituer au formateur) :** *« Pour annuaire, l'error budget de
> disponibilité est de 216 min/mois. Si on l'épuise en 2 semaines, je gèle les déploiements de
> features, je passe l'équipe en mode fiabilité (on ne shippe que des correctifs), et on ne
> rouvre le robinet qu'une fois le budget reconstitué sur la fenêtre glissante. »*

**Pièges évités :** ne pas confondre SLI (signal) et SLO (seuil) ; ne pas viser 99,99 % par défaut
(un SLO jamais tenu est inutile — *« 99,5 % tenu > 99,99 % rêvé »*) ; toujours définir la fenêtre.

> **Défi bonus (event-based vs time-based) :** pour la *disponibilité* d'annuaire, un SLO
> **event-based** (ratio de bonnes requêtes / total) est plus adapté qu'un SLO **time-based**
> (ratio de bonnes minutes) : il pondère par le trafic réel, donc une dégradation pendant un pic
> compte plus qu'une dégradation à 4h du matin. Pour la *latence* en revanche, le time-based est
> plus simple à raisonner pour de l'alerting.

---

## Étape 2 — Instrumentation Prometheus (lecture + buckets)

Code applicatif **non modifié** (interdit par le poly). Lecture du middleware
([`src/metrics.js`](services/annuaire/src/metrics.js), [`app/metrics.py`](services/planning/app/metrics.py),
[`cmd/metrics.go`](services/notif/cmd/metrics.go)) : chaque service expose
`http_requests_total` (Counter), `http_request_duration_seconds` (Histogram, **buckets via env
`METRICS_BUCKETS`**), `<svc>_build_info` (Gauge=1). Labels `method`/`route`/`status_class`,
`route` en *pattern* (`/students/:id`) et `status_class` **groupé** (`2xx…5xx`) → pas d'explosion
de cardinalité.

**Buckets choisis (alignés aux SLO de latence), via `chart/values.yaml > metrics.buckets` :**

| Service | SLO p95 | Buckets | Pourquoi |
|---|---|---|---|
| annuaire | < 300 ms | `0.05,0.1,0.2,0.3,0.5,1,2` | bucket **exact à 0.3** pour un p95 précis au seuil ; on retire `5` (jamais atteint) |
| planning | < 400 ms | `0.05,0.1,0.2,0.3,0.4,0.5,1,2` | résolution fine **autour de 0.3-0.5** (service critique) |
| notif | < 500 ms | `0.05,0.1,0.2,0.3,0.5,1,2` | bucket à 0.5 = seuil ; progression quasi-log |

> Pour annuaire : *« mes buckets sont 50/100/200/300/500 ms, 1 s, 2 s, parce que mon SLO p95 est
> 300 ms : il me faut un bucket pile à 300 ms pour que `histogram_quantile(0.95, …)` soit exact à
> ce niveau, et une progression logarithmique au-delà. »*

**`[À EXÉCUTER + CAPTURE]`** — exposition `/metrics` avec buckets effectifs :
```bash
docker run -p 8080:8080 -e METRICS_BUCKETS="0.05,0.1,0.2,0.3,0.5,1,2" ghcr.io/<vous>/annuaire:dev
curl -s localhost:8080/students >/dev/null
curl -s localhost:8080/metrics | grep http_request_duration_seconds_bucket
promtool check metrics < <(curl -s localhost:8080/metrics)   # ne doit rien signaler
```
> Coller ici la sortie montrant les `le="0.3"`, `le="0.5"`… correspondant aux buckets configurés.

**Pièges évités :** buckets **cumulatifs** (`le="0.3"` compte aussi `<0.1` et `<0.2`) ;
requêtes en `status_class!~"5.."` (et **pas** `status!~"5.."`) ; on a activé `businessEnabled: true`
(défi bonus → `business_event_total{kind="list_students"}` apparaît dans `/metrics`).

---

## Étape 3 — kube-prometheus-stack via ArgoCD

Pattern *« Argo gère sa stack d'observabilité »* : tout passe par une Application Helm.

**Livrables :**
- [`platform-sre/apps/observability/kube-prometheus-stack.yaml`](platform-sre/apps/observability/kube-prometheus-stack.yaml) — version **figée** `targetRevision: 65.5.0` (pas de HEAD mouvant), pattern multi-sources (`$values`).
- [`platform-sre/values/kube-prometheus-stack-values.yaml`](platform-sre/values/kube-prometheus-stack-values.yaml).

**Décisions de values :**
- **Désactivé sur kind** : `kubeControllerManager`, `kubeScheduler`, `kubeEtcd`, `kubeProxy`
  (control-plane non exposée par kind → cibles DOWN permanentes) et
  `prometheusOperator.admissionWebhooks` (auto-patch qui bloque en local).
- **Sélection permissive** : `serviceMonitorSelector` / `ruleSelector` = `release: kps`
  (cross-namespace), `…SelectorNilUsesHelmValues: false`.
- **Mot de passe Grafana SORTI DE GIT** (corrige le `changeme` initial) : `grafana.admin.existingSecret:
  grafana-admin`. Secret créé hors-Git — voir
  [`platform-sre/secrets/grafana-admin.example.yaml`](platform-sre/secrets/grafana-admin.example.yaml).

**AppProject élargi** ([`platform-sre/projects/devhub.yaml`](platform-sre/projects/devhub.yaml)) :
`clusterResourceWhitelist` complétée avec `ClusterRole`, `ClusterRoleBinding`,
`Mutating/ValidatingWebhookConfiguration` (sinon la stack reste OutOfSync à la création des CRDs/RBAC).

**`[À EXÉCUTER + CAPTURE]`** :
```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin --from-literal=admin-password='<motdepasse-fort>'
kubectl apply -f platform-sre/bootstrap/root-app.yaml      # bootstrap app-of-apps
kubectl get pods -n monitoring                              # tous Running ?
```
> Capture UI ArgoCD : `kube-prometheus-stack` **Synced + Healthy**. Capture
> Prometheus → Status → Targets : kube-state-metrics, node-exporter, l'API K8s **UP**.

> **Défi bonus (méta-observabilité) :** scraper Prometheus par lui-même est utile pour alerter sur
> `prometheus_tsdb_*`, `up{job="prometheus"} == 0`, ou un `scrape_duration_seconds` qui explose.
> Alerter sur « un Prometheus qui ne scrape plus » nécessite **un second œil** (Alertmanager
> dédié, ou un dead-man's-switch type `Watchdog` toujours-firing dont l'absence déclenche une alerte
> côté outil externe).

---

## Étape 4 — ServiceMonitor + dashboard Grafana

**ServiceMonitor** templatisé dans chaque chart (`templates/servicemonitor.yaml`), activé par
`monitoring.enabled` (values-dev). Points clés :
- Label **`release: kps`** sur le ServiceMonitor (sinon Prometheus l'ignore — **piège n°1 du poly**).
- Le `spec.selector` cible le **Service générique** (qui porte `release: kps`) et non les Services
  stable/canary (qui partagent les mêmes `selectorLabels`) → on évite un scraping en triple.
- **Relabelings** : on remonte `rollouts_pod_template_hash` (distinction canary/stable, indispensable
  étape 7) et un label **`app`** stable (= nom du chart) pour filtrer par service sans dépendre du
  nom de release.

**Dashboard Grafana** — choix d'un **dashboard générique réutilisable** piloté par les variables
`$datasource`, `$namespace`, `$app` (un seul dashboard couvre les 3 services), commité en JSON
dans [`platform-sre/dashboards/devhub-red.json`](platform-sre/dashboards/devhub-red.json) et chargé
en GitOps via un mini-chart (`Files.Glob` → ConfigMap `grafana_dashboard=1` → sidecar Grafana).
Plus aucun ré-import manuel après redéploiement de Grafana.

**Les 4 panneaux et leur PromQL :**
```promql
# 1. Request rate (RPS) par route — le "R" de RED
sum(rate(http_requests_total{app="$app", namespace="$namespace"}[5m])) by (route)

# 2. Error rate (% 5xx) — le "E" de RED ; seuil rouge à 1 %
sum(rate(http_requests_total{app="$app", namespace="$namespace", status_class="5xx"}[5m]))
/ clamp_min(sum(rate(http_requests_total{app="$app", namespace="$namespace"}[5m])), 1)

# 3. Latence p50/p95/p99 — le "D" de RED ; agrégation by (le) AVANT le quantile
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{app="$app", namespace="$namespace"}[5m])) by (le))

# 4. Build info — quel tag d'image est actif (table)
{__name__=~".+_build_info", app="$app", namespace="$namespace"}
```
*Explications :* (1) sert le SLI throughput ; (2) sert le SLO de disponibilité (seuil 1 %) ;
(3) sert le SLO de latence (p95) ; (4) répond à *« quelle version tourne réellement ? »* (SLI fraîcheur).

**`[À EXÉCUTER + CAPTURE]`** : Prometheus → Targets → votre service **UP=1** ;
`rate(http_requests_total[5m])` non nul après quelques `curl` ; dashboard Grafana affichant du trafic.

**Pièges évités :** label `release` (cf. supra) ; `sum(rate(...)) by (le)` **avant** `histogram_quantile`
(sinon quantile par instance, illisible) ; `rate` (lisse) et non `irate` (pics) pour un dashboard.

> **Défi bonus :** PrometheusRules écrites (étape 10), validables `promtool check rules`. Un panneau
> *« error budget consommé »* se calcule par `1 - (sum(increase(http_requests_total{status_class!~"5.."}[30d])) / sum(increase(http_requests_total[30d])))` rapporté au budget `(1-SLO)`.

---

## Étape 5 — Du Deployment au Rollout (annuaire, canary)

**Installation Argo Rollouts** via Application ArgoCD figée
([`platform-sre/apps/observability/argo-rollouts.yaml`](platform-sre/apps/observability/argo-rollouts.yaml),
chart `2.37.7`, dashboard activé sur `rollouts.devhub.local`).

**Migration annuaire :** `Deployment → Rollout` ([`templates/rollout.yaml`](services/annuaire/chart/templates/rollout.yaml)).
Le Deployment de secours **n'est plus rendu** (`useDeployment: false`) — pas de doublon de ReplicaSet.
Deux Services [`stable`](services/annuaire/chart/templates/service-stable.yaml) /
[`canary`](services/annuaire/chart/templates/service-canary.yaml) ; l'ingress pointe sur le **stable**,
Argo Rollouts gère le split via `trafficRouting.nginx.stableIngress`.

**Steps (état initial étape 5, sans Analysis) :** `setWeight 20 → pause 30s → 50 → pause 30s → 100`.

**`[À EXÉCUTER + CAPTURE]`** :
```bash
kubectl get crd | grep rollouts                       # CRDs posées ?
# Déclencher un canary : changer le tag d'image (commit) puis :
kubectl argo rollouts get rollout annuaire-dev-annuaire -n devhub-dev --watch
```
> Capture du `get rollout` pendant le canary (poids courant + étape) + dashboard `rollouts.devhub.local`.

**Pièges évités :** suppression du `Deployment` (sinon 2 ReplicaSets qui se battent) ; routing par
l'ingress-controller (sinon split approximatif au nombre de réplicas) ; annotations `canary-*` gérées
par le contrôleur.

> **Défi bonus :** pendant un canary, `kubectl get rs,ingress -n devhub-dev -l app.kubernetes.io/name=annuaire`
> montre un **second Ingress** créé par le contrôleur (`…-annuaire-canary`) portant les annotations
> `nginx.ingress.kubernetes.io/canary*` — c'est lui qui réalise le pourcentage de split.

---

## Étape 6 — Canary manuel : pause / promote / abort

**Steps avec pause manuelle :** `setWeight 10 → pause {} (indéfinie) → 50 → pause 1m → 100`.

| Scénario | Commande | Observation attendue |
|---|---|---|
| **1. Promotion normale** | `kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev` | 10 % → 50 % (attend 1 min) → 100 % |
| **2. Annulation (abort)** | `kubectl argo rollouts abort annuaire-dev-annuaire -n devhub-dev` | poids canary → 0, l'ancien RS reprend tout |
| **3. Promotion forcée** | `kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev --full` | saute toutes les étapes restantes |

**`[À EXÉCUTER + CAPTURE]`** : 3 captures avant/après du dashboard, une par scénario.

**Réponse argumentée — *quand `promote --full` est-il acceptable ?*** Uniquement en **vraie urgence** :
(a) la version actuelle est déjà cassée et la nouvelle est un hotfix vérifié par ailleurs ; (b) on
sait que l'analyse n'apportera rien (ex. correctif de config trivial) ; (c) on accepte explicitement de
**troquer la sécurité du canary contre la vitesse**. Précautions : prévenir l'équipe, garder
l'historique du Rollout pour l'audit, et surveiller manuellement les dashboards juste après — car on
vient de désactiver le filet de sécurité.

**Pièges notés :** `abort` ramène le cluster à l'ancienne image mais **ne touche pas Git** → drift.
Il faut **décider** : `git revert` (aligner Git sur le cluster) ou `promote` (aligner le cluster sur Git).
`abort` ne nettoie pas l'historique (voulu : audit).

> **Défi bonus :** un `Experiment` (CRD distincte) lance deux versions en parallèle sur 1 % chacune
> **sans intention de promouvoir** → utile pour de l'**A/B test technique** (comparer deux implémentations
> sous trafic réel, mesurer, puis décider hors-bande).

---

## Étape 7 — AnalysisTemplate : la promotion sur preuve

**Livrable :** [`templates/analysistemplate.yaml`](services/annuaire/chart/templates/analysistemplate.yaml)
référencé dans un **step `analysis` entre `setWeight: 25` et `setWeight: 50`**.

```
steps: setWeight 25 → pause 30s → ANALYSIS (5 min) → setWeight 50 → pause 1m → setWeight 100
```

**Deux métriques** interrogeant `http://prometheus-operated.monitoring.svc.cluster.local:9090`,
échantillonnées **toutes les 30 s, 10 fois (= 5 min)**, `failureLimit: 1` (une mesure ratée → rollback) :

```promql
# error-rate (CANARY uniquement) < 1 %  — clamp_min évite le NaN sur trafic faible
sum(rate(http_requests_total{app="annuaire", namespace="devhub-dev",
         rollouts_pod_template_hash="{{args.canary-hash}}", status_class="5xx"}[1m]))
/ clamp_min(sum(rate(http_requests_total{app="annuaire", namespace="devhub-dev",
         rollouts_pod_template_hash="{{args.canary-hash}}"}[1m])), 1)

# latency-p95 (CANARY) < 0.3 s  — "or vector(0)" : trafic nul → 0 plutôt qu'Inconclusive
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{app="annuaire", namespace="devhub-dev",
           rollouts_pod_template_hash="{{args.canary-hash}}"}[2m])) by (le)) or vector(0)
```

**Distinction canary/stable** : le `rollouts_pod_template_hash` est remonté par le ServiceMonitor
(relabeling, étape 4) et **injecté à l'exécution** par le Rollout
(`args: canary-hash → valueFrom.podTemplateHashValue: Latest`). Sans ce filtre, on mesurerait la
moyenne canary+stable, qui dilue le signal (*« mon AnalysisTemplate retourne toujours successful »*).

> **Détail Helm important :** dans un template Helm, `{{args.canary-hash}}` est protégé par un
> backtick raw string (`{{ \`{{args.canary-hash}}\` }}`) pour rester **littéral** (templating Argo) et
> ne pas être interprété par Helm. (Validé : `helm template` ré-émet bien `{{args.canary-hash}}`.)

**Discussion des seuils & durée :** 1 % d'erreur et p95 = SLO (300 ms) sont assez stricts pour
attraper une régression évidente, assez larges pour ne pas faire échouer un rollout valide sur un pic
statistique. **5 min** (et non 1 min) car en trafic local faible, le quantile p95 manque de points sur
une fenêtre courte → risque d'`Inconclusive`/NaN.

**`[À EXÉCUTER + CAPTURE]`** :
```bash
# Cas nominal : générer du trafic propre pendant le canary
while true; do curl -s annuaire.devhub.local/students >/dev/null; sleep 0.2; done
# Cas dégradé : déployer avec FAIL_RATE > 0 (env du Rollout) et marteler /break
while true; do curl -s annuaire.devhub.local/break >/dev/null; sleep 0.2; done
kubectl argo rollouts get analysisrun <nom> -n devhub-dev
```
> Capture 1 : AnalysisRun **Successful** → promotion auto. Capture 2 : AnalysisRun **Failed** →
> Rollout **Degraded**, canary redescendu à 0 (rollback auto).

> **Défi bonus :** 3ᵉ métrique de **saturation CPU** du canary via
> `container_cpu_usage_seconds_total` (cAdvisor/kubelet) ; et `AnalysisTemplate` paramétrable
> (`service`, `namespace`, `errorThreshold`) réutilisable par plusieurs services à seuils différents.

---

## Étape 8 — Blue/Green (planning)

**Livrable :** planning migré en `strategy.blueGreen`
([`templates/rollout.yaml`](services/planning/chart/templates/rollout.yaml)) :
`activeService` / `previewService`, `autoPromotionEnabled: false`, `scaleDownDelaySeconds: 300`,
puis `prePromotionAnalysis` (2 min de vérif preview avant bascule). Le `previewService` est exposé
sur `planning-preview.devhub.local`
([`templates/ingress-preview.yaml`](services/planning/chart/templates/ingress-preview.yaml)) — **sinon
il est inaccessible** (piège du poly).

**`[À EXÉCUTER + CAPTURE]`** :
```bash
kubectl get pods -n devhub-dev -l app.kubernetes.io/name=planning   # 2× plus de pods pendant la bascule
curl planning-preview.devhub.local/slots                            # nouvelle version (preview)
curl planning.devhub.local/slots                                    # encore l'ancienne (active)
kubectl argo rollouts promote planning-dev-planning -n devhub-dev   # bascule
```
> Capture d'une bascule manuelle réussie (active passe à la nouvelle version, ancien RS conservé 5 min).

### Schéma comparatif canary vs blue/green

| Critère | **Canary** (annuaire) | **Blue/Green** (planning) |
|---|---|---|
| Trafic pendant le déploiement | fraction progressive (5→100 %) | 0 % nouvelle (preview) puis **bascule unique** |
| Ressources | ~1× (+ qq pods canary) | **2×** (deux stacks complètes) |
| Rollback | redescendre le poids | **instantané** (rebascule) |
| Granularité de détection | fine (par paliers) | binaire (avant/après bascule) |
| Cas d'usage | service à fort trafic, régression progressive | release atomique, schéma à valider en entier avant exposition |

> **Défi bonus — quand blue/green plutôt que canary ?** (1) **Migration de schéma/contrat** où l'on
> veut tester *l'intégralité* de la nouvelle version d'un bloc avant de l'exposer (un canary à 5 %
> exposerait des utilisateurs à un état incohérent). (2) **Bascule réglementaire/transactionnelle**
> où l'on ne tolère pas deux versions servant simultanément (cohérence stricte). — Brancher un test
> E2E comme `prePromotionAnalysis` : `provider.job` (Kubernetes Job) ou `provider.web` (webhook)
> qui lance un script d'assertions sur N endpoints du `previewService` et renvoie succès/échec.

---

## Étape 9 — Routage avancé par header

**Livrable :** `trafficRouting.nginx.additionalIngressAnnotations` du Rollout annuaire (activé par
`rollout.canaryByHeader`) ajoute `canary-by-header: X-Beta-User` / `canary-by-header-value: "true"`.

**`[À EXÉCUTER + CAPTURE]`** :
```bash
curl annuaire.devhub.local/students                          # -> version STABLE
curl -H "X-Beta-User: true" annuaire.devhub.local/students   # -> version CANARY
```
*Usage métier :* l'équipe produit teste **chaque release sur ses propres comptes** avant n'importe quel
utilisateur. Combiné à l'`AnalysisTemplate` : on valide d'abord en interne par header, puis on ouvre le
canary pondéré au public sous surveillance Prometheus.

**Piège noté :** avec `canary-by-header`, le **header gagne** sur le `canary-weight` (ce n'est pas
additif). **Défi bonus :** `canary-by-cookie` route le client de façon **persistante** (le même
utilisateur reste sur la nouvelle version d'une requête à l'autre) → meilleure expérience pour une
vraie bêta utilisateurs qu'un header à repositionner à chaque appel.

---

## Étape 10 — Alerting Alertmanager + notifications Rollouts

**PrometheusRules** (une par service, `templates/prometheusrule.yaml`), gradation **page/ticket**,
chaque alerte avec `for:` (anti-flapping) et `runbook_url` :
- `…HighErrorRate` → **severity: page** (réveille, `for: 5m`).
- `…HighLatencyP95` → **severity: ticket** (attend lundi, `for: 30m`).
- `…TargetDown` → **severity: page** (le service ne répond plus).

**Alertmanager** ([`kube-prometheus-stack-values.yaml > alertmanager.config`](platform-sre/values/kube-prometheus-stack-values.yaml)) :
`severity=page → page-webhook` (repeat 1h), `severity=ticket → ticket-webhook` (repeat 12h),
`inhibit_rules` (si on *page*, pas de *ticket* en doublon). Mock via `webhook.site`.

**Notifications Argo Rollouts** ([`argo-rollouts-values.yaml > notifications`](platform-sre/values/argo-rollouts-values.yaml)) :
rollout **Promoted** → webhook *success*, **Aborted** → webhook *failure*. Payload : nom du Rollout,
namespace, révision, image, conclusion. Les Rollouts s'y abonnent via annotations
`notifications.argoproj.io/subscribe.*`.

**`[À EXÉCUTER + CAPTURE]`** : 3 captures webhook.site — (1) charge 5 % de 5xx → page après 5 min ;
(2) ralentissement → ticket après 30 min ; (3) rollout Promoted/Aborted → notification.
```bash
promtool check rules <(helm template annuaire-dev services/annuaire/chart -f services/annuaire/chart/values-dev.yaml -s templates/prometheusrule.yaml)
```

**Pièges évités :** pas de `severity: critical` partout (sinon plus personne ne lit l'oncall) ;
`repeat_interval` explicite (sinon re-signalement toutes les 4h = spam) ; `for:` systématique
(sinon une fluctuation transitoire trigger).

> **Défi bonus :** runbooks écrits dans [`platform-sre/runbooks/`](platform-sre/runbooks/) (un par
> alerte, structure : signification / impact / diagnostic / mitigation / escalade / post-mortem),
> linkés par `runbook_url`. Intégration Slack/Discord réelle = remplacer l'URL webhook.site par un
> webhook entrant du canal + documenter la rotation du token (Secret K8s, rotation trimestrielle).

---

## Étape 11 — Matrice comparative (note /5 + justification)

| Critère | RollingUpdate natif | Argo Rollouts | Flagger |
|---|---|---|---|
| Courbe d'apprentissage | **5** — rien à apprendre, c'est le défaut K8s | **3** — nouvelle CRD + analyses à comprendre | **2** — config plus implicite, plus de magie |
| Intégration ArgoCD (GitOps) | 4 — un Deployment se synchronise trivialement | **5** — même éditeur, l'App suit le Rollout nativement | 3 — fonctionne, mais pensé d'abord pour Flux |
| Intégration Flux (GitOps) | 4 | 3 | **5** — même éditeur (FluxCD) |
| Variété des stratégies | **1** — rolling uniquement | **5** — canary, blueGreen, A/B, analysis, experiment | 4 — canary, blueGreen, A/B (moins d'objets) |
| Variété des metric providers | 0 — aucune notion de métrique | **5** — Prometheus, Datadog, NewRelic, CloudWatch, web, job… | 4 — large aussi, un cran derrière |
| UI / dashboard prêt | 1 — `kubectl rollout status` | **5** — dashboard web + plugin kubectl | 2 — pas d'UI dédiée (Grafana à monter) |
| Coût opérationnel cluster | **5** — zéro composant en plus | 3 — un contrôleur à exploiter | 3 — un contrôleur à exploiter |
| Adapté à un mesh (Istio/Linkerd) | 2 — split au nombre de réplicas | **5** — SMI/Istio/NGINX/ALB… L7 natif | **5** — pensé mesh-first |
| Communauté / fréquence release | 4 — c'est K8s | **5** — CNCF Graduated, très actif | 4 — CNCF, actif |
| Risque si le contrôleur tombe en plein canary | **5** — pas de contrôleur tiers, K8s gère | 3 — le canary se fige dans l'état courant (pas de promotion auto, mais trafic stable) | 3 — idem |

**Synthèse :** RollingUpdate gagne sur la **simplicité et le coût** ; Argo Rollouts gagne sur la
**richesse + l'UI + l'intégration ArgoCD** (notre cas) ; Flagger est l'équivalent côté **Flux/mesh-first**.
Notre choix d'Argo Rollouts est cohérent avec la chaîne ArgoCD du TP 2.

---

## Étape 12 — Synthèse : « ma chaîne est-elle production-ready ? »

### Livrable 1 — Rétrospective TP 2 → TP 3 (ressenti par ligne)

| Opération | Ressenti TP 3 |
|---|---|
| Déployer une version | Plus **lent** mais plus **serein** : la promotion attend la preuve. |
| Détecter une dégradation | **Rassurant** : l'AnalysisTemplate voit ce qu'ArgoCD ne voyait pas. |
| Faire un rollback | **Automatique** (analysis) ou une commande — fini le `git revert` en panique. |
| Limiter l'impact | **Le gros gain** : 5 % du trafic, 5 min max, au lieu de toute la flotte. |
| Savoir si le service tient son SLO | On a enfin une **réponse mesurée**, plus un ressenti. |
| Décider de promouvoir | **Sur métriques**, plus « au feeling » / « ça devrait passer ». |
| Tester sur un cohort | **Nouveau** : header-based, impossible au TP 2. |

**Deux opérations où le surcoût TP 3 n'est PAS justifié pour une startup de 3 personnes :**
1. **Mesurer la fréquence de déploiement (DORA)** via objets Rollout + Prometheus : à 3 personnes,
   un coup d'œil aux logs CI/Git suffit ; monter la chaîne d'extraction est du sur-engineering.
2. **Header-based / A-B testing** : sans équipe produit ni cohorts à servir, la complexité du routage
   L7 n'apporte rien — un simple `RollingUpdate` + un rollback Git manuel est plus économique.
   *(Bonus : le blue/green à 2× les ressources est aussi discutable sur un petit cluster contraint.)*

**L'opération qui, seule, justifie le passage TP 2 → TP 3 dans une PME en croissance :**
**« Limiter l'impact d'une mauvaise version »** (5 % / 5 min au lieu de 100 %). C'est exactement
l'incident des 1200 emplois du temps : le coût d'un seul incident évité paie la chaîne.

### Livrable 2 — Ce que cette chaîne ne sait toujours pas faire

1. **Traçabilité distribuée** — *Risque :* un appel lent traverse plusieurs services, on voit la
   latence agrégée mais pas **où** elle naît. *Outil :* **OpenTelemetry + Jaeger/Tempo**.
   *Réf :* doc OpenTelemetry « Traces ».
2. **Logs centralisés & corrélés** — *Risque :* en incident, on saute de pod en pod pour lire les
   logs, sans lien avec la métrique. *Outil :* **Loki + Fluent Bit**, corrélation par **exemplars**.
   *Réf :* doc Grafana Loki « LogQL ».
3. **Mesure côté utilisateur (RUM)** — *Risque :* p95 serveur vert mais navigateur lent (réseau,
   JS, CDN). *Outil :* **RUM / Web Vitals** (Grafana Faro, etc.). *Réf :* web.dev « Core Web Vitals ».
4. **Chaos engineering** — *Risque :* on ne sait pas comment le service réagit à un pod tué/une
   latence injectée tant qu'on ne l'a pas essayé. *Outil :* **Chaos Mesh / LitmusChaos / Pumba**.
   *Réf :* doc Chaos Mesh.
5. **Politique d'admission** — *Risque :* un dev peut déployer un Rollout **sans** AnalysisTemplate
   et contourner tout le filet. *Outil :* **Kyverno / OPA Gatekeeper** (policy « no Rollout without
   Analysis »). *Réf :* doc Kyverno « Policies ».
6. **Signature & provenance des images** — *Risque :* rien ne garantit que l'image promue est bien
   celle buildée par la CI (supply chain). *Outil :* **Sigstore (cosign), in-toto, SLSA**.
   *Réf :* doc Sigstore.
7. **Backup applicatif & DR** — *Risque :* nos services sont in-memory ; un vrai service stateful
   n'aurait ni sauvegarde ni plan de reprise. *Outil :* **Velero, snapshots PVC, dumps SGBD**.
   *Réf :* doc Velero.

### Livrable 3 — Position d'architecte (10 services, 30 devs)

> Je **garde** la colonne vertébrale : ArgoCD + Helm + Argo Rollouts en canary par défaut, Prometheus
> + Alertmanager avec SLO et alerting gradé page/ticket. C'est ce qui transforme « on déploie » en
> « on déploie en mesurant ». Je **remplace** le mot de passe Grafana hors-Git par des **SealedSecrets/
> External Secrets** (à 30 devs, la rotation manuelle ne tient pas), et le webhook.site par une vraie
> intégration Slack + runbooks. J'**ajoute** en priorité une **policy Kyverno** « pas de Rollout sans
> AnalysisTemplate » (la rigueur ne doit pas dépendre de la discipline de chacun), puis le **tracing
> OpenTelemetry** (à 10 services, la latence agrégée ne suffit plus pour diagnostiquer). Le blue/green,
> je le réserve aux migrations atomiques ; partout ailleurs, le canary mesuré est le meilleur
> compromis coût/sécurité.

---

## Boss final (facultatif)

Méthode d'astreinte : reproduire le **symptôme**, remonter la chaîne *cause → effet* (Targets
Prometheus → ServiceMonitor/label `release` → AnalysisRun → route Alertmanager), **réparer par
commit Git uniquement** (pas de `kubectl edit` — c'est tout l'intérêt du GitOps), puis expliquer
quelle hypothèse a été écartée et pourquoi celle retenue collait. *(À jouer en direct avec le formateur.)*

---

## Annexe — Ordre de déploiement

```bash
make tools-check-tp3
make cluster-up && make argocd-install        # (réutilise le cluster TP 2 si présent)
make hosts-print                              # ajouter les *.devhub.local au fichier hosts
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin --from-literal=admin-password='<fort>'
kubectl apply -f platform-sre/bootstrap/root-app.yaml   # app-of-apps -> tout le reste
```
Ordre logique : observabilité (étapes 3-4) **avant** livraison progressive (étapes 5+) — *on ne
livre pas progressivement ce qu'on ne mesure pas*.
