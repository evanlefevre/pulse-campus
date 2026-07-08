# GUIDE DÉBUTANT — TP3 de A à Z

> Pour quelqu'un qui découvre Kubernetes. On va **tout** faire dans l'ordre. Avance phase par
> phase, et **valide chaque phase** (section ✅ Vérifier) avant la suivante. En cas d'erreur,
> regarde la **Phase 9 — Dépannage** ou demande-moi.
>
> 🐚 **Quel terminal ?** Ouvre **Git Bash** dans le dossier `pulse-campus` (clic droit → *Git Bash
> Here*). Toutes les commandes ci-dessous sont prévues pour Git Bash. Pour installer des outils avec
> `choco`, ouvre **PowerShell en tant qu'administrateur**.

---

## 0. Ce que tu vas construire (en français simple)

Imagine une petite plateforme web interne (« DevHub Campus ») avec **3 services** : `annuaire`
(liste d'étudiants), `planning` (créneaux), `notif` (notifications). Le but du TP n'est PAS de coder
ces services (ils sont déjà écrits), mais d'apprendre à **les déployer sans casser la production**.

### Le vocabulaire minimal

| Mot | En une phrase |
|---|---|
| **Conteneur / image** | Une appli empaquetée avec tout ce qu'il lui faut pour tourner. L'**image** est le modèle figé, le **conteneur** est une instance qui tourne. On les fabrique avec **Docker**. |
| **Kubernetes (K8s)** | Un chef d'orchestre qui fait tourner tes conteneurs, les redémarre s'ils tombent, les expose au réseau. |
| **Pod** | La plus petite unité K8s : un (ou quelques) conteneur(s) qui tournent ensemble. |
| **Cluster** | L'ensemble des machines gérées par Kubernetes. Ici, un **faux cluster local** créé par **kind** (Kubernetes-in-Docker) : tout tourne dans Docker sur ton PC. |
| **Helm** | Le « gestionnaire de paquets » de K8s. Un **chart** Helm = un dossier de modèles (`templates/`) + des réglages (`values.yaml`). |
| **ArgoCD** | L'outil **GitOps** : il **lit ton dépôt Git** et s'assure que le cluster correspond exactement à ce qui est écrit dans Git. Tu ne fais plus `kubectl apply` à la main : **tu commit, ArgoCD déploie.** |
| **Prometheus** | Collecte des **métriques** (nb de requêtes, erreurs, temps de réponse) en interrogeant `/metrics` de chaque service. |
| **Grafana** | Affiche ces métriques dans des **dashboards** (graphiques). |
| **Argo Rollouts** | Remplace le déploiement classique de K8s par un déploiement **progressif** : il envoie d'abord un petit % du trafic sur la nouvelle version, mesure, et **promeut ou annule (rollback)** selon les métriques. |
| **Canary** | Stratégie : 25 % → 50 % → 100 % du trafic, par paliers, avec mesure entre chaque. |
| **Blue/Green** | Stratégie : la nouvelle version tourne en parallèle (invisible), on bascule **tout d'un coup** quand elle est validée. |

### L'idée centrale du TP

> Au TP2, ArgoCD disait « le cluster = le Git, tout est vert » — mais ne disait PAS si le service
> **fonctionnait vraiment**. Un déploiement a cassé 1200 emplois du temps sans que rien ne s'allume
> en rouge. Le TP3 ajoute la **preuve par la mesure** : on ne promeut une version que si Prometheus
> confirme qu'elle ne dégrade ni le taux d'erreur ni la latence.

### Qui vit où dans le cluster

- **namespace `argocd`** → ArgoCD (le déployeur GitOps)
- **namespace `monitoring`** → Prometheus + Grafana + Alertmanager
- **namespace `argo-rollouts`** → le contrôleur Argo Rollouts (+ son dashboard web)
- **namespace `devhub-dev`** → tes 3 services (annuaire en canary, planning en blue/green, notif classique)
- **namespace `ingress-nginx`** → la porte d'entrée HTTP (route `*.devhub.local` vers les services)

---

## 1. Installer les outils (Windows)

Tu as déjà : **Docker, kubectl, helm, git**. Il manque : **kind** (obligatoire),
**kubectl-argo-rollouts** (obligatoire pour le TP), **argocd** et **promtool** (optionnels).

### 1.1 Docker Desktop doit tourner et avoir assez de RAM
- Lance **Docker Desktop**, attends qu'il soit « running ».
- ⚙️ Settings → Resources : donne-lui **au moins 6 Go de RAM** (la stack de monitoring est lourde).

### 1.2 kind (obligatoire) — dans **PowerShell admin**
```powershell
choco install kind -y
choco install make -y      # pratique : permet d'utiliser les raccourcis "make ..."
```
Vérifie (dans Git Bash) : `kind version`

### 1.3 kubectl-argo-rollouts (obligatoire) — téléchargement manuel
C'est un **plugin** de kubectl. Dans **PowerShell** :
```powershell
New-Item -ItemType Directory -Force C:\kube-tools | Out-Null
Invoke-WebRequest -Uri https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64 -OutFile C:\kube-tools\kubectl-argo-rollouts.exe
# Ajoute C:\kube-tools au PATH (permanent) :
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\kube-tools", "User")
```
**Ferme et rouvre** ton terminal, puis vérifie : `kubectl argo rollouts version`

### 1.4 argocd CLI (optionnel — l'UI web suffit) — PowerShell
```powershell
Invoke-WebRequest -Uri https://github.com/argoproj/argo-cd/releases/latest/download/argocd-windows-amd64.exe -OutFile C:\kube-tools\argocd.exe
```

### 1.5 promtool (optionnel — pour valider les règles d'alerte) — PowerShell
Va sur https://prometheus.io/download/, télécharge la version **windows-amd64**, dézippe, et copie
`promtool.exe` dans `C:\kube-tools`.

### ✅ Vérifier
Dans Git Bash, depuis le dossier `pulse-campus` :
```bash
make tools-check-tp3
```
(Si `make` n'est pas installé, lance les commandes `kind version`, `kubectl argo rollouts version`
une par une.)

---

## 2. Mettre le projet sur GitHub + adapter les réglages

> 🧠 **Point crucial à comprendre :** ArgoCD ne lit PAS tes fichiers locaux. Il lit un **dépôt
> GitHub**. Donc : tout ce que tu modifies en local doit être **commité et poussé** sur GitHub pour
> qu'ArgoCD le voie. « Modifier en local sans pousser » = ArgoCD ne voit rien.

### 2.1 Crée un compte + un dépôt GitHub
1. Compte sur https://github.com (si tu n'en as pas).
2. Crée un dépôt **public** nommé `pulse-campus` (public = ArgoCD y accède sans mot de passe, plus
   simple pour débuter). **Ne** coche **pas** « Add a README ».

### 2.2 Pousse le code (dans Git Bash, depuis `pulse-campus`)
Remplace `TONHANDLE` par ton identifiant GitHub :
```bash
git init
git add .
git commit -m "TP3 : squelette + manifestes"
git branch -M main
git remote add origin https://github.com/TONHANDLE/pulse-campus.git
git push -u origin main
```
(Au push, GitHub demande tes identifiants : utilise un **Personal Access Token** comme mot de passe
— GitHub → Settings → Developer settings → Tokens (classic) → cocher `repo`.)

### 2.3 Remplace les placeholders dans tout le projet
Deux chaînes à remplacer partout (remplace `TONHANDLE` par ton identifiant GitHub) :
```bash
# 1) l'URL de ton dépôt (apparaît dans les manifestes ArgoCD + les runbook_url)
grep -rl 'evanlefevre' . | xargs sed -i "s/evanlefevre/TONHANDLE/g"

# 2) le compte des images Docker (ghcr.io/changeme/... -> ghcr.io/TONHANDLE/...)
grep -rl 'changeme' . | xargs sed -i "s/changeme/TONHANDLE/g"
```
> 💡 Tu n'as **pas besoin de pousser les images sur GHCR** : on les chargera directement dans kind
> (Phase 4). On garde juste le nom `ghcr.io/TONHANDLE/<svc>` par cohérence.

### 2.4 Re-pousse
```bash
git add -A && git commit -m "config: mon handle + mon compte d'images" && git push
```

### ✅ Vérifier
`grep -r 'evanlefevre' .` ne doit **rien** renvoyer. Pareil pour `grep -r 'changeme' .`
(sauf éventuellement le `Makefile`, sans importance).

---

## 3. Créer le cluster + ingress + ArgoCD

### 3.1 Crée le cluster kind (2 nœuds)
```bash
make cluster-up
# équivaut à : kind create cluster --name pulse --config cluster/kind-config.yaml
```

### 3.2 Installe ingress-nginx + ArgoCD
```bash
make argocd-install
```
> Ça prend quelques minutes (téléchargements). Si `make` n'existe pas, copie le bloc `argocd-install:`
> du `Makefile` et lance ses commandes une à une dans Git Bash.

### 3.3 Édite le fichier hosts (pour accéder aux `*.devhub.local`)
Affiche les lignes à ajouter :
```bash
make hosts-print
```
Ouvre **Notepad en administrateur**, ouvre le fichier
`C:\Windows\System32\drivers\etc\hosts`, et colle :
```
127.0.0.1  argocd.devhub.local
127.0.0.1  grafana.devhub.local
127.0.0.1  prometheus.devhub.local
127.0.0.1  rollouts.devhub.local
127.0.0.1  annuaire.devhub.local
127.0.0.1  planning.devhub.local
127.0.0.1  notif.devhub.local
```

### 3.4 Récupère le mot de passe admin d'ArgoCD + ouvre l'UI
```bash
# mot de passe admin initial :
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d ; echo
# accès à l'UI (laisse cette commande tourner dans un terminal dédié) :
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Ouvre **https://localhost:8080** (accepte l'avertissement TLS), login `admin` + le mot de passe.

### ✅ Vérifier
```bash
kubectl get pods -n argocd          # tous Running
kubectl get pods -n ingress-nginx   # controller Running
```

---

## 4. Construire et charger les images des 3 services

> On build les images **localement** et on les **injecte dans kind** (`kind load`). Comme la
> politique de pull est `IfNotPresent` et que l'image est déjà présente, Kubernetes ne va PAS
> essayer de la télécharger depuis Internet. **Aucun compte GHCR nécessaire.**

Remplace `TONHANDLE` :
```bash
docker build -t ghcr.io/TONHANDLE/annuaire:dev services/annuaire
docker build -t ghcr.io/TONHANDLE/planning:dev services/planning
docker build -t ghcr.io/TONHANDLE/notif:dev    services/notif

kind load docker-image ghcr.io/TONHANDLE/annuaire:dev --name pulse
kind load docker-image ghcr.io/TONHANDLE/planning:dev --name pulse
kind load docker-image ghcr.io/TONHANDLE/notif:dev    --name pulse
```

### ✅ Vérifier
```bash
docker exec pulse-control-plane crictl images | grep TONHANDLE   # les 3 images apparaissent
```

---

## 5. Déployer toute la plateforme (le bootstrap GitOps)

### 5.1 Crée le namespace monitoring + le secret Grafana
(Grafana refuse de démarrer sans ce secret — on le crée AVANT.)
```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='ChangeMoi123!'
```

### 5.2 Lance l'app-of-apps (la SEULE chose qu'on applique à la main)
```bash
kubectl apply -f platform-sre/bootstrap/root-app.yaml
```
À partir de là, **ArgoCD prend le relais** : l'application `root` lit `platform-sre/apps/` sur ton
GitHub et crée toutes les autres applications (monitoring, rollouts, dashboards, et tes 3 services).

### ✅ Vérifier (patience : le 1er déploiement pèse plusieurs Go)
- Dans l'UI ArgoCD (https://localhost:8080) : les applications passent en **Synced + Healthy**.
- En ligne de commande :
```bash
kubectl get applications -n argocd
kubectl get pods -n monitoring     # prometheus, grafana, alertmanager, node-exporter, kube-state-metrics
kubectl get pods -n devhub-dev     # annuaire, planning, notif
kubectl get pods -n argo-rollouts  # le contrôleur
```
> 🟠 Si une app reste **OutOfSync/Progressing** longtemps : c'est souvent normal au début (CRDs).
> Clique « Sync » dans l'UI si besoin. Voir Phase 9 si ça bloque.

---

## 6. Voir l'observabilité fonctionner

### 6.1 Génère un peu de trafic
```bash
for i in $(seq 1 200); do
  curl -s annuaire.devhub.local/students > /dev/null
  curl -s planning.devhub.local/slots   > /dev/null
  curl -s notif.devhub.local/events     > /dev/null
done
```

### 6.2 Prometheus
Ouvre **http://prometheus.devhub.local** → onglet **Status → Targets** : tes 3 services doivent être
**UP**. Puis onglet **Graph**, tape :
```promql
sum(rate(http_requests_total{app="annuaire"}[5m]))
```

### 6.3 Grafana
Ouvre **http://grafana.devhub.local** → login `admin` / `ChangeMoi123!`. Menu **Dashboards** →
**« DevHub Campus — RED par service »**. En haut, choisis `namespace=devhub-dev` et `app=annuaire`.
Tu vois RPS, taux d'erreur, latences p50/p95/p99, et la version active.

### 6.4 Dashboard Argo Rollouts
Ouvre **http://rollouts.devhub.local** : tu vois tes Rollouts `annuaire` (canary) et `planning` (blue/green).

### ✅ Vérifier
Le dashboard Grafana affiche du trafic après les `curl`. Les targets Prometheus sont UP.

---

## 7. Jouer la livraison progressive (le cœur du TP)

### 7.1 Canary annuaire — promotion automatique (cas nominal, étape 7)
On déclenche un nouveau déploiement. Le plus simple **sans rebuild** : changer une variable
d'environnement dans les values, ce qui crée une nouvelle version de pod.

1. Édite `services/annuaire/chart/values-dev.yaml`, change `LOG_LEVEL: debug` en `LOG_LEVEL: info`.
2. Commit + push :
   ```bash
   git add -A && git commit -m "annuaire: trigger canary" && git push
   ```
3. ArgoCD synchronise (ou clique Sync). Pendant ce temps, observe :
   ```bash
   kubectl argo rollouts get rollout annuaire-dev-annuaire -n devhub-dev --watch
   ```
4. En parallèle, **génère du trafic** (sinon l'analyse manque de points) :
   ```bash
   while true; do curl -s annuaire.devhub.local/students > /dev/null; sleep 0.2; done
   ```
Le canary monte à 25 %, pause, **lance une AnalysisRun** (interroge Prometheus 5 min), puis 50 %,
puis 100 % automatiquement si les métriques sont bonnes.

### 7.2 Canary annuaire — rollback automatique (cas dégradé, étape 7)
On simule une régression : la nouvelle version renvoie des erreurs 500.

1. Édite `services/annuaire/chart/values.yaml`, sous `env:` ajoute une ligne `FAIL_RATE: "0.5"`.
2. Commit + push, laisse ArgoCD synchroniser.
3. Génère du trafic **sur la route qui casse** :
   ```bash
   while true; do curl -s annuaire.devhub.local/break > /dev/null; sleep 0.1; done
   ```
L'AnalysisRun voit le taux de 5xx dépasser 1 % → passe en **Failed** → le Rollout **rollback**
(canary redescend à 0, l'ancienne version reprend tout). 🎉 C'est la promesse du TP.
> N'oublie pas de **remettre `FAIL_RATE: "0"`** (commit/push) après la démo.

### 7.3 Piloter à la main (étape 6)
```bash
kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev        # avancer d'un palier
kubectl argo rollouts abort   annuaire-dev-annuaire -n devhub-dev        # annuler (retour stable)
kubectl argo rollouts promote annuaire-dev-annuaire -n devhub-dev --full # tout promouvoir d'un coup (risqué)
```

### 7.4 Blue/Green planning (étape 8)
Déclenche un déploiement de planning (ex. change `LOG_LEVEL` dans `services/planning/chart/values-dev.yaml`, push).
```bash
kubectl get pods -n devhub-dev -l app.kubernetes.io/name=planning   # 2x plus de pods pendant la bascule
curl planning-preview.devhub.local/slots    # la NOUVELLE version (preview), pas encore exposée aux users
curl planning.devhub.local/slots            # encore l'ANCIENNE (active)
kubectl argo rollouts promote planning-dev-planning -n devhub-dev   # bascule l'active vers la nouvelle
```

### 7.5 Routage par header (étape 9)
```bash
curl annuaire.devhub.local/students                          # version stable
curl -H "X-Beta-User: true" annuaire.devhub.local/students   # version canary (pendant un canary en cours)
```

---

## 8. Les captures pour le RAPPORT.md

Le fichier [`RAPPORT.md`](RAPPORT.md) est déjà rédigé. Il te reste à **coller tes captures** aux
endroits marqués `[À EXÉCUTER + CAPTURE]`. Checklist :
- [ ] sortie de `kubectl argo rollouts version` + `promtool --version` (étape 0)
- [ ] `curl /metrics` montrant tes buckets (étape 2)
- [ ] UI ArgoCD : kube-prometheus-stack **Synced + Healthy** (étape 3)
- [ ] Prometheus Targets UP + dashboard Grafana avec trafic (étape 4)
- [ ] `get rollout` pendant un canary (étape 5)
- [ ] 3 captures promote/abort/--full (étape 6)
- [ ] AnalysisRun **Successful** (promotion auto) + **Failed** (rollback auto) (étape 7)
- [ ] bascule blue/green réussie (étape 8)
- [ ] les 2 `curl` header stable vs canary (étape 9)
- [ ] 3 webhooks reçus (page / ticket / rollout) (étape 10) — voir note ci-dessous

> **Webhooks (étape 10)** : crée 3 URLs sur https://webhook.site, et remplace les
> `REMPLACEZ-...` dans `platform-sre/values/kube-prometheus-stack-values.yaml` (Alertmanager) et
> `platform-sre/values/argo-rollouts-values.yaml` (notifications). Commit + push.

---

## 9. Dépannage (erreurs fréquentes)

| Symptôme | Cause probable / solution |
|---|---|
| Pod en `ImagePullBackOff` | Image pas chargée dans kind. Refais la Phase 4 (`docker build` + `kind load`), puis `kubectl delete pod <pod> -n devhub-dev` pour forcer le recréation. |
| `Prometheus ne voit pas mon service` (target absente) | Label `release: kps` manquant sur le Service, ou `monitoring.enabled: false`. Vérifie `kubectl get svc -n devhub-dev --show-labels`. |
| App ArgoCD **OutOfSync** au 1er sync | Souvent les CRDs du kube-prometheus-stack. Clique **Sync** dans l'UI ; au pire, Sync avec l'option *Replace*. |
| `histogram_quantile` renvoie **NaN** | Pas assez de trafic / de points. Lance la boucle de `curl` plus longtemps. |
| Canary monte à 100 % en 2 secondes | Pas de trafic → l'analyse passe trop vite, ou `pause` mal réglée. Génère du trafic AVANT. |
| AnalysisRun toujours **Successful** même en cas d'erreur | La requête PromQL ne filtre pas le canary. (Déjà géré ici via `rollouts_pod_template_hash`.) |
| Grafana ne démarre pas | Le secret `grafana-admin` n'existe pas dans le namespace `monitoring`. Refais la Phase 5.1. |
| `*.devhub.local` ne répond pas | Fichier `hosts` non édité (Phase 3.3), ou le port 80 est pris par une autre appli sur Windows. |
| ArgoCD ne voit pas mes changements | Tu as oublié de **commit + push**. ArgoCD lit GitHub, pas tes fichiers locaux. |

---

## 10. Tout éteindre / nettoyer

```bash
make cluster-down          # supprime le cluster kind (équiv. kind delete cluster --name pulse)
```
Tes fichiers et ton dépôt GitHub restent intacts. Pour tout reprendre : recommence à la Phase 3.

---

### Récapitulatif de l'ordre

1. Installer (Phase 1) → 2. GitHub + placeholders (Phase 2) → 3. Cluster + ArgoCD (Phase 3) →
4. Build + load images (Phase 4) → 5. Bootstrap (Phase 5) → 6. Vérifier l'observabilité (Phase 6) →
7. Jouer les canaries / blue-green (Phase 7) → 8. Captures rapport (Phase 8).

**Règle d'or GitOps :** *je modifie → je commit → je push → ArgoCD déploie.* Jamais de `kubectl edit`
pour « réparer » : on corrige dans Git.
