# Runbooks — DevHub Campus SRE

Chaque alerte `PrometheusRule` porte une annotation `runbook_url` qui pointe vers
un fichier de ce dossier. Un runbook utile **à 3h du matin** contient au minimum :

1. **Signification** — ce que dit concrètement l'alerte (pas la requête PromQL brute).
2. **Impact** — qui/quoi est touché, à quel point c'est urgent.
3. **Diagnostic** — les 3-4 commandes à lancer en premier (dashboards, `kubectl`, PromQL).
4. **Mitigation** — l'action qui arrête l'hémorragie (souvent : `kubectl argo rollouts abort` / `git revert`).
5. **Escalade** — qui prévenir si ça ne suffit pas.
6. **Post-mortem** — quoi noter pour éviter la récidive.

Convention : un fichier par alerte, nommé `<service>-<slug-alerte>.md`.
