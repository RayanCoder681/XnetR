# XnetR

**Automatisation RPA d'activation de forfaits Internet — sans API, sans modification du système existant.**

XnetR pilote un dashboard manager existant exactement comme le ferait un opérateur humain : connexion, recherche d'utilisateur, sélection de profil, activation de forfait, confirmation du résultat. Le tout orchestré par un moteur Playwright, exposé via une API interne, et journalisé de bout en bout.

---

## Pourquoi XnetR

Le système réseau cible ne fournit **aucune API publique**. Toutes les activations de forfaits passent par une interface graphique pensée pour un usage manuel, poste par poste — lent, répétitif, sujet à l'erreur humaine, difficile à auditer.

XnetR automatise ce parcours sans jamais toucher au système existant : il reproduit les actions d'un manager via navigation automatisée (Playwright), et expose le tout comme un service interne fiable.

```
activatePackage("client01", "1GB_DAY", 5)
```

## Fonctionnement

| Étape | Description |
|---|---|
| 1. Connexion | Ouverture d'une session manager sécurisée et persistée |
| 2. Recherche | Localisation du compte utilisateur cible |
| 3. Sélection | Choix du profil Internet et de la quantité |
| 4. Activation | Validation de l'activation sur le dashboard |
| 5. Confirmation | Lecture du résultat et journalisation complète |

Les activations sont traitées **une par une** par session manager (concurrency = 1) pour éviter tout conflit d'état sur le système cible.

## Architecture

```
Client
  │
  ▼
Frontend XnetR (Next.js)
  │  REST / WebSocket
  ▼
Backend API (NestJS + Fastify)
  │  enqueue job
  ▼
Redis + BullMQ (file d'attente, concurrency = 1)
  │
  ▼
Automation Worker (Playwright, pattern Page Object)
  │  actions navigateur
  ▼
Dashboard existant (jamais modifié)
```

L'API ne pilote jamais Playwright directement : elle délègue à un worker isolé via la file d'attente. Ça isole les pannes navigateur, permet de scaler chaque brique indépendamment, et limite l'impact d'un changement d'interface côté dashboard au seul worker.

## Stack technique

| Composant | Choix | Pourquoi |
|---|---|---|
| Backend | NestJS + Fastify | ~2x plus léger qu'Express |
| ORM | Drizzle ORM | Pas de binaire natif embarqué (contrairement à Prisma) |
| Base de données | PostgreSQL | Fiabilité transactionnelle |
| File d'attente | Redis + BullMQ | Concurrency control natif, retries, dead-letter queue |
| Automatisation | Playwright | API moderne, pattern Page Object |
| Frontend | Next.js + TypeScript | Écosystème partagé avec le backend |
| Package manager | pnpm | Empreinte disque réduite |
| Conteneurisation | Docker multi-image | API sur `node:22-alpine`, worker sur `playwright:jammy` (Playwright incompatible Alpine) |

## Modèle de données

- `clients` — comptes XnetR autorisés à utiliser le système
- `manager_sessions` — état des sessions Playwright authentifiées
- `activation_jobs` — chaque demande d'activation (statut, payload, résultat)
- `audit_logs` — trace exhaustive de chaque action effectuée sur le dashboard

## Sécurité

- Identifiants manager chiffrés (AES-256), jamais exposés au frontend ni journalisés en clair
- Rotation automatique des sessions en cas d'expiration
- Authentification des clients XnetR par JWT
- Journalisation complète de toute action sur le système existant

## Gestion des erreurs

| Erreur | Stratégie |
|---|---|
| Utilisateur introuvable | Échec immédiat, pas de retry |
| Session expirée | Re-authentification automatique, puis reprise du job |
| Interface changée (sélecteur introuvable) | Échec, capture d'écran automatique, alerte |
| Timeout réseau/DOM | Retry avec backoff exponentiel (3 tentatives max) |
| Activation échouée côté dashboard | Échec, log détaillé, vérification humaine |

## Démarrage rapide

### Prérequis

- Node.js ≥ 20
- pnpm ≥ 9
- Docker + Docker Compose

### Installation

```bash
git clone <repo>
cd xnetr
pnpm install
```

### Configuration

```bash
# apps/api/.env
DATABASE_URL=postgres://user:password@localhost:5432/xnetr
REDIS_URL=redis://localhost:6379
JWT_SECRET=change-me

# apps/worker/.env
DATABASE_URL=postgres://user:password@localhost:5432/xnetr
REDIS_URL=redis://localhost:6379
DASHBOARD_URL=https://dashboard-existant.example.com
MANAGER_USERNAME=xxxxx      # à chiffrer/stocker via un secret manager en prod
MANAGER_PASSWORD=xxxxx
```

⚠️ Ne jamais committer de vrais identifiants. En production, ces valeurs doivent provenir d'un gestionnaire de secrets (Vault, AWS Secrets Manager, etc.).

### Lancer l'infrastructure locale

```bash
docker compose up -d postgres redis
pnpm --filter api drizzle-kit migrate
```

### Développement

```bash
pnpm --filter api dev       # API — http://localhost:3000
pnpm --filter worker dev    # Worker Playwright
pnpm --filter web dev       # Frontend — http://localhost:3001
```

### Tests

```bash
pnpm test              # unitaires
pnpm test:integration   # intégration API
pnpm test:e2e           # end-to-end Playwright
```

## Utiliser l'API

```bash
curl -X POST http://localhost:3000/activations \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{ "username": "client01", "profile": "1GB_DAY", "quantity": 5 }'
```

```bash
curl http://localhost:3000/activations/job_8f2a... \
  -H "Authorization: Bearer <token>"
```

## Déploiement

```bash
docker compose -f docker-compose.prod.yml up -d --build
```

- Image `api` : `node:22-alpine` (légère)
- Image `worker` : `mcr.microsoft.com/playwright:v1.5x-jammy` (obligatoire, Playwright n'est pas compatible Alpine)

## Roadmap

- [x] Architecture — schéma technique, choix de stack, modèle de données
- [ ] Backend — API NestJS, modules, DTOs
- [ ] Automation Engine — Page Objects Playwright, `activatePackage`
- [ ] Frontend — interface client Next.js
- [ ] Tests — unitaires, intégration, end-to-end
- [ ] Déploiement — Dockerisation, CI/CD, monitoring

**Bloquant actuel :** obtenir les sélecteurs CSS/XPath réels du dashboard existant (connexion, recherche, sélection de profil, bouton d'activation, zone de résultat) pour finaliser le worker.

## Documentation

- `XnetR-Cahier-des-charges.md` — spécification complète du projet
- `PROMPT-CURSOR.md` — prompt de génération pour Cursor (Agent mode)
- `index.html` — landing page de présentation

## Licence

Projet interne — usage restreint.
