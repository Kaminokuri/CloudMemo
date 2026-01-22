# CloudMemo (Flask + Redis) — Conteneurisation Docker/Podman

Projet **Flask + Redis** conteneurisé, orchestré avec **docker-compose**.  
Pour l’instant, l’objectif est de faire tourner une app web simple qui **incrémente un compteur `hits` dans Redis** et l’affiche sur la page `/`.

> ✅ Fait : Dockerfile + docker-compose + build d’image + tests + push (optionnel)  
> 🧱 À venir : Gunicorn (prod), Kubernetes, CI/CD, monitoring (templates prêts à remplir)

---

## Sommaire

- [Aperçu](#aperçu)
- [Architecture](#architecture)
- [Structure du projet](#structure-du-projet)
- [Pré-requis](#pré-requis)
- [Démarrage rapide](#démarrage-rapide)
- [Dockerfile](#dockerfile)
- [docker-compose](#docker-compose)
- [Variables d’environnement](#variables-denvironnement)
- [Problèmes rencontrés et résolutions](#problèmes-rencontrés-et-résolutions)
- [Publication sur GitHub](#publication-sur-github)
- [Roadmap](#roadmap)
- [Licence](#licence)

---

## Aperçu

### Endpoint

- `GET /`

### Comportement

- Connexion à Redis via `REDIS_HOST`
- Incrémente `hits`
- Répond par exemple :

```

CloudMemo: Hello World! I have been seen 42 times.

```

Si Redis est indisponible :

```

CloudMemo: Redis is not reachable.

````

---

## Architecture

- **app** : Flask (port `5000`)
- **redis** : Redis (port interne `6379`)
- **docker-compose** : lance 2 services et connecte l’app à Redis via le nom de service `redis`

---

## Structure du projet

> (à ajuster si besoin)

```txt
.
├── app.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
````

---

## Pré-requis

* Docker **ou** Podman
* docker-compose (ou `podman-compose`)
* Accès réseau pour récupérer `redis:alpine`

---

## Démarrage rapide

### 1) Lancer

```bash
docker-compose up -d
```

### 2) Tester

* Dans le navigateur : `http://localhost:5000`
* Ou en CLI :

```bash
curl http://localhost:5000
```

### 3) Arrêter

```bash
docker-compose down
```

---

## Dockerfile

Dockerfile utilisé :

```dockerfile
# Utiliser une image Python légère
FROM python:3.9-slim

# Définir le répertoire de travail dans le conteneur
WORKDIR /app

# Copier les dépendances (le fichier est juste ici, pas dans app/)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copier tout le reste du dossier actuel dans le conteneur
COPY . .

# Exposer le port
EXPOSE 5000

# Lancer l'app
CMD ["python", "app.py"]
```

Build :

```bash
docker build -t cloudmemo:v1 .
# ou
podman build -t cloudmemo:v1 .
```

---

## docker-compose

`docker-compose.yml` utilisé :

```yaml
version: '3'
services:
  app:
    image: cloudmemo:v1
    ports:
      - "5000:5000"
    environment:
      - REDIS_HOST=redis
  redis:
    image: redis:alpine
```

---

## Variables d’environnement

* `REDIS_HOST` : hôte Redis

  * en local sans compose : `localhost`
  * en compose : `redis` (nom du service)

---

## Problèmes rencontrés et résolutions

### 1) Flask affiche : “This is a development server…”

**Symptôme :**

* Warning dans les logs Flask indiquant que ce n’est pas fait pour la prod.

**Cause :**

* L’app est lancée via `python app.py` → serveur de dev Flask.

**Résolution (actuelle) :**

* Accepté pour un POC / validation du fonctionnement.

**Amélioration prévue :**

* Remplacer par **gunicorn** (voir roadmap).

---

### 2) Redis non joignable (`Redis is not reachable`)

**Cause :**

* Redis pas démarré, ou variable `REDIS_HOST` incorrecte.

**Résolution :**

* En `docker-compose`, utiliser :

  * `REDIS_HOST=redis`
* Vérifier les conteneurs :

  ```bash
  docker ps
  docker-compose logs -f
  ```

---

### 3) Podman + docker-compose : besoin du socket Podman

**Contexte :**

* Utilisation de Podman tout en pilotant via des commandes “Docker”/`docker-compose`.

**Résolution appliquée :**

```bash
systemctl --user enable --now podman.socket
export DOCKER_HOST=unix:///run/user/$(id -u)/podman/podman.sock
docker-compose up
```

---

### 4) Push registry / login : galères d’auth + (mauvaise) solution temporaire

**Symptômes typiques :**

* échecs de login/push
* tentatives répétées d’auth
* utilisation de `--tls-verify=false` pour “débloquer”

**Résolution appliquée (court terme) :**

* Login avec `--password-stdin` (mieux que taper le mot de passe en clair)
* Nettoyage des fichiers d’auth Podman si nécessaire

**Important (sécurité) :**

* ⚠️ Évite de laisser traîner des tokens (PAT) dans l’historique shell ou dans des fichiers.
  Si un token a été affiché/stocké, **révoque-le et régénère-en un** côté DockerHub, puis utilise `--password-stdin`.

Exemple plus propre :

```bash
export DOCKERHUB_TOKEN="********"
printf '%s' "$DOCKERHUB_TOKEN" | podman login docker.io -u <user> --password-stdin
unset DOCKERHUB_TOKEN
```

---

### 5) Logs système (non bloquants pour le projet)

* `vmwgfx ... [drm] *ERROR*` : lié au driver graphique (VM), sans impact direct sur l’app
* `PAM unable to dlopen(pam_lastlog.so)` : module PAM manquant (système)
* `The user 'sudo' does not exist.` : groupe `sudo` absent selon la distro (parfois `wheel`)
