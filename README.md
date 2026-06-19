# CESIZENdeploy

Repo de déploiement centralisé pour le projet CESIZEN — application de bien-être mental développée dans le cadre du cursus CESI.

![CI Status](https://github.com/Plabou102/CESIZENdeploy/actions/workflows/cicd.yml/badge.svg)

## Architecture

Le projet est composé de 3 repos :
- **CESIZENAPI** — API REST Express/TypeScript avec Prisma et PostgreSQL
- **CESIZENReact** — Frontend React/Vite
- **CESIZENdeploy** — CI/CD et configuration de déploiement (ce repo)

L'application est accessible sur [https://cesizenadame.duckdns.org](https://cesizenadame.duckdns.org)

---

## CI/CD Pipeline

### Déclenchement

La pipeline se déclenche sur :
- Push et Pull Request sur `CESIZENdeploy`
- `repository_dispatch` depuis `CESIZENAPI` et `CESIZENReact` via un workflow `notify-deploy.yml` dans chaque repo

### Jobs

| Job | Description | Branche |
|-----|-------------|---------|
| `build-lint-test` | Build, lint et tests unitaires | Toutes |
| `audit` | Audit de sécurité des dépendances (`npm audit`) | Toutes |
| `gitleaks` | Scan de secrets dans le code | Toutes |
| `super-linter` | Analyse statique multi-langage | Toutes |
| `build-and-push-docker` | Build et push des images sur ghcr.io | `main` |
| `deploy` | Déploiement sur la VM Azure via SSH | `main` |

---

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

---

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
- Mots de passe hachés (bcrypt) — jamais stockés en clair
- Authentification JWT (Bearer token)
- CORS configuré — origines autorisées via variable d'environnement
- HTTPS obligatoire via Traefik + Let's Encrypt
- Validation des entrées avec Zod (protection injection SQL)
- Prisma ORM — protection native contre les injections SQL

---

## Matrice des risques

| Vulnérabilité | Probabilité | Impact | Criticité | Action mise en place |
|---|---|---|---|---|
| Brute force SSH | 3 | 3 | **9 — Critique** | Fail2ban, clé SSH uniquement |
| Secrets dans le code | 2 | 3 | **6 — Élevée** | Gitleaks CI, variables d'env |
| Injection SQL | 2 | 3 | **6 — Élevée** | Prisma ORM, validation Zod |
| Vol de token JWT | 2 | 3 | **6 — Élevée** | HTTPS, expiration courte |
| Dépendances vulnérables | 3 | 2 | **6 — Élevée** | npm audit dans la CI |
| Accès non autorisé API | 2 | 3 | **6 — Élevée** | JWT middleware, CORS |
| XSS | 2 | 2 | **4 — Moyenne** | React échappe les données par défaut |
| Fuite données PostgreSQL | 1 | 3 | **3 — Faible** | Port 5432 non exposé, réseau Docker interne |

*Probabilité × Impact (1=Faible, 2=Moyen, 3=Élevé)*

---

## Gestion de crise

En cas d'attaque ou incident de sécurité :

**1. Détection**
- Uptime Kuma envoie une alerte Discord si un service tombe
- Fail2ban log les tentatives bloquées
- Logs Docker accessibles via `docker logs [conteneur]`

**2. Containment**
- Couper immédiatement la VM Azure depuis le portail
- Révoquer tous les tokens et secrets GitHub Actions
- Changer le mot de passe de la base de données

**3. Analyse**
- Analyser les logs SSH : `sudo journalctl -u ssh`
- Analyser les logs Traefik : `docker logs traefik`
- Identifier l'origine de l'attaque

**4. Remédiation**
- Ne jamais nettoyer une machine compromise — reconstruire depuis zéro
- Regénérer tous les secrets (JWT_SECRET, DB_PASSWORD, GH_PAT, SSH_KEY)
- Redéployer depuis une image propre via la CI/CD

**5. Post-mortem**
- Documenter ce qui s'est passé
- Identifier la faille exploitée
- Mettre en place les correctifs pour éviter la récidive

---

## RGPD

L'application CESIZEN collecte des données personnelles des utilisateurs.

**Données collectées :**
- Identité : prénom, nom, email
- Données d'usage : historique des exercices de respiration, vues d'informations

**Mesures techniques :**
- Mots de passe hachés avec bcrypt — jamais stockés en clair
- HTTPS obligatoire — données chiffrées en transit
- Accès à la base de données restreint au réseau Docker interne
- Authentification JWT — seul l'utilisateur peut accéder à ses données
- Données minimales collectées — pas de données inutiles

**Droits des utilisateurs (RGPD) :**
- Droit à l'effacement (Art. 17) — possibilité de supprimer un compte
- Droit d'accès — l'utilisateur peut consulter ses données via l'API
- Pas de partage de données avec des tiers

---

## Veille technologique

**Outils intégrés dans le projet :**
- `npm audit` — vérifie les CVE dans les dépendances à chaque push
- `Gitleaks` — détecte les secrets exposés, base de signatures mise à jour régulièrement
- `Dependabot` — alertes GitHub automatiques sur les dépendances vulnérables
- Mise à jour régulière des images Docker utilisées

**Sources de veille :**
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — les 10 vulnérabilités web les plus critiques
- [ANSSI](https://www.ssi.gouv.fr/) — recommandations françaises de cybersécurité
- [CVE Database](https://cve.mitre.org/) — base publique des vulnérabilités connues
- [The Hacker News](https://thehackernews.com/) — actualités sécurité et nouvelles menaces

---

## Docker Images

Les images sont hébergées sur **GitHub Container Registry (ghcr.io)** et buildées via le CI à chaque push sur `main`.

Le Dockerfile de `CESIZENReact` utilise un **multi-stage build** :
1. **Stage builder** — installe les dépendances et compile avec Vite
2. **Stage final** — sert uniquement le dossier `dist` avec `serve` (~50MB au lieu de ~500MB)

Un `.dockerignore` est présent dans chaque repo pour exclure `.git`, `node_modules` et `.env` des images.

---

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

---

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
