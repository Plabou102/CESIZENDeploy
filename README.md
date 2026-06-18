
# CESIZEN Deploy

Repo de déploiement centralisé pour le projet CESIZEN — application de bien-être mental développée dans le cadre du cursus CESI.

[![CI CESIZEN Deploy](https://github.com/Plabou102/CESIZENDeploy/actions/workflows/cicd.yml/badge.svg)](https://github.com/Plabou102/CESIZENDeploy/actions/workflows/cicd.yml)

## Architecture

Le projet est composé de 3 repos :
- **CESIZENAPI** — API REST Express/TypeScript avec Prisma et PostgreSQL
- **CESIZENReact** — Frontend React/Vite
- **CESIZENdeploy** — CI/CD et configuration de déploiement (ce repo)

L'application est accessible sur [https://cesizenadame.duckdns.org](https://cesizenadame.duckdns.org)

## CI/CD Pipeline

![CI Status](https://github.com/Plabou102/CESIZENdeploy/actions/workflows/cicd.yml/badge.svg)

### Déclenchement

La pipeline se déclenche sur :
- Push et Pull Request sur `CESIZENdeploy`
- `repository_dispatch` depuis `CESIZENAPI` et `CESIZENReact` via un workflow `notify-deploy.yml` dans chaque repo

### Jobs

| Job | Description | Branche |
|-----|-------------|---------|
| `build-lint-test-e2e` | Build, lint, tests unitaires et end to end | Toutes |
| `audit` | Audit de sécurité des dépendances (`npm audit`) | Toutes |
| `gitleaks` | Scan de secrets dans le code | Toutes |
| `super-linter` | Analyse statique multi-langage | Toutes |
| `build-and-push-docker` | Build et push des images sur ghcr.io | `main` |
| `deploy` | Déploiement sur la VM Azure via SSH | `main` |

## Infrastructure

### VM Azure
- **OS** : Ubuntu 24.04 LTS
- **Provider** : Microsoft Azure (abonnement étudiant)
- **Accès** : SSH par clé uniquement

### Docker
Les services tournent dans des conteneurs Docker gérés par Docker Compose :

| Conteneur | Image | Description |
|-----------|-------|-------------|
| `traefik` | `traefik:v2.11` | Reverse proxy |
| `cesizen-api` | `ghcr.io/plabou102/cesizen-api:main` | API REST |
| `cesizen-react` | `ghcr.io/plabou102/cesizen-react:main` | Frontend |
| `cesizen-db` | `postgres:16` | Base de données |
| `uptime-kuma` | `louislam/uptime-kuma:1` | Monitoring |

### Traefik
Traefik est le reverse proxy qui gère le routage HTTP/HTTPS :
- Routing par domaine et préfixe de chemin
- Certificats SSL automatiques via **Let's Encrypt**
- Accessible sur `cesizenadame.duckdns.org`

### Monitoring
**Uptime Kuma** monitore la disponibilité des services :
- Frontend : `https://cesizenadame.duckdns.org`
- API : `https://cesizenadame.duckdns.org/api/health`
- Notifications Discord en cas de panne

## Sécurité

### Infrastructure
- Connexion SSH par clé uniquement (pas de mot de passe)
- **Fail2ban** installé pour bloquer les tentatives SSH abusives (5 tentatives max, ban 1h)
- Ports ouverts : 80, 443, 22 uniquement
- Variables d'environnement pour tous les secrets (pas de credentials en dur)

### CI/CD
- **Gitleaks** — détection de secrets dans le code source
- **Super-Linter** — analyse statique du code
- **npm audit** — audit des dépendances Node.js
- Secrets GitHub Actions pour SSH, tokens et mots de passe

### Application
- Mots de passe hachés (bcrypt)
- Authentification JWT (Bearer token)
- CORS configuré
- HTTPS obligatoire via Traefik + Let's Encrypt

## Docker Images

Les images sont hébergées sur **GitHub Container Registry (ghcr.io)** et buildées via le CI à chaque push sur `main`.

Le Dockerfile de `CESIZENReact` utilise un **multi-stage build** :
1. **Stage builder** — installe les dépendances et compile avec Vite
2. **Stage final** — sert uniquement le dossier `dist` avec `serve`

Un `.dockerignore` est présent dans chaque repo pour exclure `.git`, `node_modules` et `.env` des images.

## Variables d'environnement

Les secrets sont injectés via GitHub Actions Secrets au moment du déploiement :

| Variable | Description |
|----------|-------------|
| `DB_PASSWORD` | Mot de passe PostgreSQL |
| `JWT_SECRET` | Clé secrète JWT |
| `CORS_ORIGIN` | Origine autorisée pour CORS |
| `SEED_PASSWORD` | Mot de passe pour le seed |
| `SSH_KEY` | Clé privée SSH |
| `SSH_HOST` | IP de la VM |
| `SSH_USER` | Utilisateur SSH |
| `GH_PAT` | Personal Access Token GitHub |

## Lancer en local

```bash
# Cloner les repos
git clone https://github.com/Plabou102/CESIZENdeploy.git
git clone https://github.com/Plabou102/CESIZENReact.git
git clone https://github.com/Plabou102/CESIZENAPI.git

# Lancer l'API
cd CESIZENAPI
cp .env.example .env
npm install
npm run dev

# Lancer le frontend
cd CESIZENReact
npm install
npm run dev
```
