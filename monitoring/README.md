# Monitoring — Metrics & Alerts

Ce document explique les métriques exposées par l'application, la règle d'alerte proposée et les étapes pour tester et déployer Prometheus / Grafana.

**Chemins importants dans le dépôt**

- Dashboard Grafana (provisioning) : `monitoring/grafana/provisioning/dashboards/cv_dashboard.json`
- Exemple d'instrumentation (si présent) : `monitoring/instrumentation/metrics.py`
- Règles Prometheus proposées : `monitoring/prometheus/rules/low_confidence_alerts.yml`


## 1) Objectif

Fournir des métriques Prometheus pour :
- Suivre le nombre total de prédictions (`cv_predictions_total`).
- Suivre les prédictions à faible confiance (`cv_low_confidence_predictions_total`).
- Mesurer le temps d'inférence (`cv_inference_time_seconds`).
- Indiquer si l'application voit la base de données (`cv_database_connected`).

Ces métriques permettent de calculer le "Low Confidence Ratio" (part des prédictions à faible confiance) et d'alerter si ce ratio devient trop élevé.


## 2) Métriques exposées (exemples)

- `cv_predictions_total` (Counter) : incrémenté après chaque prédiction.
- `cv_low_confidence_predictions_total` (Counter) : incrémenté pour chaque prédiction dont la confiance est considérée faible (ex. max(proba_cat, proba_dog) < 0.7 — seuil configurable).
- `cv_inference_time_seconds` (Histogram) : durée des inférences.
- `cv_database_connected` (Gauge) : 1 si l'application peut se connecter à la DB, 0 sinon.

Exemple d'instrumentation Python (prometheus_client) :

```python
from prometheus_client import Counter, Gauge, Histogram

cv_predictions_total = Counter('cv_predictions_total', 'Nombre total de prédictions')
cv_low_confidence_predictions_total = Counter('cv_low_confidence_predictions_total', 'Predictions with low confidence')
cv_inference_time_seconds = Histogram('cv_inference_time_seconds', 'Inference time in seconds')
cv_database_connected = Gauge('cv_database_connected', '1 if DB reachable else 0')

# usage example inside the app
with cv_inference_time_seconds.time():
    proba_cat, proba_dog = model.predict_proba(image)
cv_predictions_total.inc()
if max(proba_cat, proba_dog) < 0.7:
    cv_low_confidence_predictions_total.inc()

# set DB gauge
cv_database_connected.set(1 if db_is_ok else 0)
```


## 3) Calcul du "Low Confidence Ratio" (Prometheus)

Prometheus / Grafana utilise l'expression :

```
100 * sum(increase(cv_low_confidence_predictions_total[5m])) / sum(increase(cv_predictions_total[5m]))
```

- `increase(...[5m])` calcule le nombre d'événements durant la fenêtre glissante de 5 minutes.
- `sum(...)` permet d'agréger si tu as plusieurs instances.
- On multiplie par 100 pour obtenir un pourcentage.

Variante (utilisant `rate`):

```
100 * sum(rate(cv_low_confidence_predictions_total[5m])) / sum(rate(cv_predictions_total[5m]))
```

Conseil : pour éviter les faux positifs sur peu de trafic, combine avec une condition de volume minimal :

```
(sum(increase(cv_predictions_total[5m])) > 10) and (100 * sum(increase(cv_low_confidence_predictions_total[5m])) / sum(increase(cv_predictions_total[5m])) > 20)
```


## 4) Requête SQL équivalente (Postgres)

Si tu veux calculer le même indicateur depuis la table `predictions_feedback` :

```sql
SELECT
  100.0 * SUM(CASE WHEN GREATEST(proba_cat, proba_dog) < 0.7 THEN 1 ELSE 0 END) / NULLIF(COUNT(*), 0) AS low_confidence_pct,
  COUNT(*) AS total_predictions
FROM predictions_feedback
WHERE created_at >= now() - interval '5 minutes';
```

Ajuste `0.7` pour le seuil de "low confidence".


## 5) Règle d'alerte Prometheus (exemple)

Fichier d'exemple : `monitoring/prometheus/rules/low_confidence_alerts.yml`

Règle proposée :
- Alerter si plus de 20% des prédictions des 10 dernières minutes sont low-confidence et si au moins 20 prédictions ont eu lieu.
- Alerter si la DB est injoignable (cv_database_connected == 0) pendant 2 minutes.

Exemple d'expression (PromQL) :

```
# ratio > 20% sur 10m
100 * sum(increase(cv_low_confidence_predictions_total[10m])) / sum(increase(cv_predictions_total[10m])) > 20

# faible volume
sum(increase(cv_predictions_total[10m])) < 20

# db inaccessible
avg_over_time(cv_database_connected[2m]) == 0
```


## 6) Où placer les fichiers et comment déployer

- Place les règles sous le dossier chargé par Prometheus (`prometheus.yml` doit inclure `rule_files:` pointant vers les fichiers `.yml`).
- Si tu utilises Docker Compose (stack fournie dans le dépôt), tu peux généralement copier `monitoring/prometheus/rules/*.yml` vers le répertoire monté dans le conteneur Prometheus ou lier le dossier.

Commandes courantes (depuis le dossier du repo) :

```powershell
# redémarrer la stack docker
docker compose down ; docker compose up -d --build

# voir les logs Prometheus
docker compose logs prometheus --tail 200

# voir les logs Grafana
docker compose logs grafana --tail 200
```

Si Prometheus est sur un VPS et que tu modifes des fichiers sur l'hôte, redémarre le service Prometheus ou envoie une requête de reload :

```powershell
# from host where prometheus runs
docker kill -s HUP prometheus_container_name
# or restart
docker restart prometheus_container_name
```


## 7) Tester les métriques

- Dans Prometheus (interface web), tester les expressions :
  - `increase(cv_predictions_total[5m])`
  - `increase(cv_low_confidence_predictions_total[5m])`
  - `100 * sum(increase(cv_low_confidence_predictions_total[5m])) / sum(increase(cv_predictions_total[5m]))`

- Dans Grafana → Explore, tester la même expression et vérifier qu'elle retourne des valeurs.


## 8) Dépannage fréquent

- Pas de données : l'application n'expose peut-être pas `/metrics` ou le port n'est pas scrappé par Prometheus.
  - Vérifier que l'endpoint `/metrics` est accessible et contient les métriques listées.
  - Vérifier le scrape config de Prometheus (targets).

- Grafana ne peut pas joindre Postgres : si Grafana tourne dans Docker, utilise le nom du service (`postgres:5432`) et non `localhost:5437` (5437 est souvent le port mappé sur l'hôte).

- Ratio `NaN` ou vide : probablement peu ou pas de prédictions dans la fenêtre. Utiliser un seuil minimal de volume.


## 9) Je peux t'aider à :
- Intégrer ces fichiers dans ton `docker-compose.yml` (montage des règles Prometheus, variables d'environnement pour Grafana).
- Ajouter un panneau SQL dans Grafana pour afficher la requête SQL fournie.
- Générer une règle d'alerte Grafana si tu préfères gérer les alertes via Grafana Alerting au lieu de Prometheus.

Dis-moi ce que tu veux que j'ajoute ou modifie dans ce `README.md` ou si tu veux que je crée les fichiers de règles dans `monitoring/prometheus/rules/` (je peux les ajouter automatiquement).