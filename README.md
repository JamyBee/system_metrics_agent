# System Metrics Agent

Un agent Python modulaire capable de collecter les métriques système,
de les formater et de les transmettre en temps réel vers une API ou un
Webhook externe.

## Fonctionnalités

- Collecte du pourcentage CPU.
- Collecte de la mémoire RAM totale, utilisée et disponible.
- Utilisation de `psutil`.
- Utilisation de **`subprocess`** pour exécuter la commande système
  `uptime` et récupérer la charge système.
- Architecture modulaire :
  - `collector.py` : collecte ;
  - `formatter.py` : normalisation du payload ;
  - `sender.py` : envoi HTTP ;
  - `agent.py` : orchestration ;
  - `api.py` : API de réception avec **FastAPI**.
- Configuration externe avec `.env`.
- Gestion des erreurs réseau et des erreurs de collecte.
- Tests automatisés avec `pytest`.

## Architecture

```text
system_metrics_agent/
├── app/
│   ├── __init__.py
│   ├── agent.py
│   ├── api.py
│   ├── collector.py
│   ├── config.py
│   ├── formatter.py
│   └── sender.py
├── tests/
│   ├── test_api.py
│   ├── test_collector.py
│   ├── test_formatter.py
│   └── test_sender.py
├── .env.example
├── requirements.txt
└── README.md
```

## Installation

Créer un environnement virtuel :

```bash
python -m venv .venv
```

Linux/macOS :

```bash
source .venv/bin/activate
```

Windows :

```powershell
.venv\Scripts\Activate.ps1
```

Installer les dépendances :

```bash
pip install -r requirements.txt
```

Créer le fichier de configuration :

```bash
cp .env.example .env
```

Sous Windows, copier manuellement `.env.example` en `.env`.

## Démarrer l'API FastAPI

```bash
uvicorn app.api:app --reload
```

L'API est alors disponible sur :

```text
http://127.0.0.1:8000
```

Documentation interactive FastAPI :

```text
http://127.0.0.1:8000/docs
```

## Démarrer l'agent

Dans un deuxième terminal :

```bash
python -m app.agent
```

L'agent collecte les métriques selon l'intervalle défini dans `.env`
puis les envoie automatiquement vers `METRICS_ENDPOINT`.

## Exemple de configuration

```env
METRICS_ENDPOINT=http://127.0.0.1:8000/metrics
COLLECTION_INTERVAL=5
REQUEST_TIMEOUT=5
```

Pour envoyer vers un webhook externe, remplacer simplement :

```env
METRICS_ENDPOINT=https://example.com/webhook
```

## Exemple de métrique envoyée

```json
{
  "agent": "system-metrics-agent",
  "event_type": "system_metrics",
  "data": {
    "timestamp": "2026-08-27T12:00:00.000000+00:00",
    "hostname": "server-01",
    "cpu": {
      "percent": 23.4,
      "logical_cores": 8
    },
    "memory": {
      "total_bytes": 16777216000,
      "available_bytes": 8000000000,
      "used_bytes": 7777216000,
      "percent": 48.2
    },
    "system": {
      "load_1m": 0.12,
      "load_5m": 0.18,
      "load_15m": 0.20
    }
  }
}
```

## Endpoints

### Vérifier l'état de l'API

```http
GET /health
```

Réponse :

```json
{
  "status": "ok"
}
```

### Envoyer des métriques

```http
POST /metrics
```

### Récupérer la dernière métrique

```http
GET /metrics/latest
```

## Tests

Exécuter toute la suite :

```bash
pytest -q
```

Exécuter avec une couverture de code nécessite l'installation de
`pytest-cov` :

```bash
pip install pytest-cov
pytest --cov=app --cov-report=term-missing
```

## Robustesse

Le projet gère notamment :

- les erreurs d'exécution de `subprocess` ;
- les timeouts ;
- les erreurs de connexion HTTP ;
- les réponses HTTP en erreur ;
- les configurations invalides ;
- les payloads incomplets ;
- l'absence de métriques dans l'API.

## Démonstration du flux

```text
┌──────────────────┐
│ Système          │
│ CPU / RAM        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ collector.py     │
│ psutil           │
│ subprocess       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ formatter.py     │
│ Payload JSON     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ sender.py        │
│ HTTP POST        │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ FastAPI          │
│ /metrics         │
└──────────────────┘
```


---

# Rapport TP DevOps (INGC2)

## Choix techniques

- **Image de base** : `python:3.12-slim-bookworm` (légère, version précise, jamais `latest`). Alpine écartée : musl complique l'installation de `psutil` et peut allonger les builds.
- **Multi-stage** : un étage `builder` installe les dépendances ; l'image finale ne reçoit que le résultat (`COPY --from=builder`), sans outils de build.
- **Dépendances séparées** : `requirements-api.txt` (FastAPI, uvicorn) et `requirements-agent.txt` (psutil, requests, python-dotenv), versions épinglées. pytest, black et flake8 ne sont pas dans les images de production.
- **Cache et taille** : dépendances copiées avant le code, `pip install --no-cache-dir`, `.dockerignore` (.git, .venv, .env, tests).
- **Sécurité** : utilisateur non-root (`appuser`), aucun secret dans l'image ni dans le dépôt (`.env` ignoré, `.env.example` fourni), secrets GitHub pour Docker Hub et SonarQube Cloud.
- **Prêt pour la prod** : uvicorn sans `--reload`, `--host 0.0.0.0`, `EXPOSE 8000`, `HEALTHCHECK` sur `/health` (via Python, `curl` étant absent de l'image slim).
- **Commande `uptime`** : absente de l'image slim, fournie par le paquet Debian `procps`, ajouté uniquement à l'image de l'agent.
- **Docker Compose** : l'agent joint l'API par son nom de service (`http://api:8000/metrics`), démarre quand l'API est saine (`service_healthy`), configuration via `.env`, `restart: unless-stopped`.

## Tailles des images

| Image | Taille sur disque | Taille du contenu |
|---|---|---|
| metrics-api | 221 MB | 53 MB |
| metrics-agent | 2026 MB | 49.4 MB |

Images publiées : https://hub.docker.com/r/<TON-NOM-DOCKERHUB>/metrics-api et https://hub.docker.com/r/jamesbee007/metrics-agent

## Pipeline (GitHub Actions)

1. **test** : pytest avec couverture, analyse SonarQube Cloud, Quality Gate bloquant.
2. **build-scan-push** (matrice api/agent, après `test`) : build local, scan Trivy (HIGH/CRITICAL corrigeables, `exit-code: 1`), puis push Docker Hub (tags SHA et `latest`) uniquement sur `main`. L'image poussée est celle qui a été scannée. L'action Trivy est épinglée par commit.

## Preuve d'échec volontaire

J'ai remplacé `requests==2.34.2` par `requests==2.19.1` (version vulnérable). Trivy a échoué sur l'image de l'agent (failles HIGH sur `requests` et `urllib3`). Après fusion dans `main`, l'étape de push a été ignorée et le tag `latest` sur Docker Hub est resté inchangé. En revenant à `requests==2.34.2`, le pipeline est repassé au vert et l'image saine a été publiée.

## Conteneur et hôte

Dans un conteneur, `psutil` lit les informations du noyau de l'hôte (ici la machine virtuelle de Docker Desktop) et non les limites du conteneur. Le nom d'hôte relevé est l'identifiant du conteneur, pas celui du serveur. Pour surveiller un vrai serveur, il faut donc soit installer l'agent directement sur l'hôte, soit lui donner accès au système hôte (réseau et `/proc` de l'hôte) ; pour surveiller un conteneur, lire ses limites cgroup.

## Difficultés rencontrées

- `pyyaml` ne s'installait pas sous Python 3.15 (pas de version précompilée) : passage à Python 3.12, aligné sur l'image Docker.
- 2 tests échouaient sous Windows (la fonction `get_load_average` renvoie `None` hors Linux) : tests rendus indépendants de l'OS.
- Le README annonce un dossier `tests/`, mais les tests sont dans `app/tests/`.
- Un workflow hérité du projet d'origine (`pipeline.yaml`, avec déploiement sur un VPS tiers) échouait et faisait doublon : supprimé.
- Avertissements non bloquants dans Actions (dépréciation de Node 20, migration d'`ubuntu-latest`).