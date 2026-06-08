# ADR 0004 — PostgreSQL and Prisma ORM

## Status

Accepted

## Context

Le projet nécessite une base de données relationnelle pour gérer des entités fortement liées (Organisation → Clients → Chantiers → Pointages → Factures). Les calculs de facturation impliquent des montants financiers qui doivent rester précis (éviter les erreurs d'arrondi des types `float`).

Les migrations de schéma doivent être contrôlées et versionnées : une migration mal appliquée en production peut causer une perte de données ou un downtime.

## Decision

La base de données est **PostgreSQL**. L'ORM est **Prisma**.

### Règles de typage des montants

- Tous les champs montants (`unit_price`, `total`, `hourly_rate`, `km_rate`, etc.) utilisent le type `Decimal` dans le schéma Prisma, mappé sur `NUMERIC` dans PostgreSQL.
- Aucun champ montant ne doit utiliser `Float` dans Prisma.
- Les calculs côté Node.js utilisent `decimal.js` ou `big.js` — jamais le type `number` natif JavaScript pour des opérations financières.
- La stratégie d'arrondi (règle suisse 0,05 CHF) et les taux de TVA applicables seront confirmés avant la Phase 4 (voir `docs/06-open-questions.md` — Q2 et Q3). Une ADR complémentaire sera créée si la logique est structurante.

### Règles de migration

- `prisma migrate dev` — développement uniquement (génère et applique les migrations localement)
- `prisma migrate deploy` — production uniquement (applique les migrations existantes sans en créer de nouvelles)
- **Jamais `prisma db push` en production** — cette commande peut entraîner des pertes de données
- Une PR = une migration au maximum
- Les fichiers de migration sont versionnés dans git et relus avant merge
- Les migrations sont testées sur une copie de la base avant déploiement en production

### Règles d'utilisation de Prisma

- Les entités Prisma restent dans l'infrastructure (`features/<name>/infrastructure/prisma/`)
- Les entités Prisma ne sont jamais exposées directement via les controllers
- Des mappers (`features/<name>/infrastructure/mappers/`) convertissent les modèles Prisma en entités de domaine et en DTOs de réponse
- Le `PrismaService` est partagé via `shared/infrastructure/`

## Alternatives considered

**TypeORM** — ORM historiquement populaire avec NestJS. Écarté car les migrations TypeORM sont moins fiables que celles de Prisma (génération automatique parfois incorrecte), et le schéma déclaratif de Prisma est plus lisible.

**Drizzle ORM** — léger, typesafe, basé sur des requêtes SQL explicites. Intéressant mais ecosystème moins mature que Prisma au moment de cette décision. À reconsidérer si Prisma pose des problèmes structurants.

**MikroORM** — bonne alternative à TypeORM, supporte mieux les patterns DDD. Moins de documentation et d'exemples disponibles que Prisma dans l'écosystème NestJS.

**MySQL / SQLite** — PostgreSQL offre des types plus riches (`NUMERIC`, `JSONB`, Row Level Security future), un meilleur support des transactions complexes et un écosystème d'hébergement CH/EU solide (Exoscale, Infomaniak). SQLite écarté car inadapté au multi-utilisateur.

## Consequences

- **Positif :** schéma Prisma lisible, migrations fiables et versionnées.
- **Positif :** `NUMERIC` PostgreSQL garantit la précision des montants financiers.
- **Positif :** PostgreSQL Row Level Security peut être ajouté plus tard comme couche de sécurité supplémentaire sans changer le schéma (voir ADR 0005).
- **Négatif :** les entités Prisma et les entités de domaine sont distinctes — les mappers ajoutent du code.
- **Vigilance :** ne jamais utiliser `prisma db push` en dehors du développement initial du schéma.
- **Vigilance :** revoir les migrations générées automatiquement avant de les committer — Prisma peut générer des opérations destructives sur les colonnes existantes.

---

*Veyra — Greder Montagen · ADR 0004 · Juin 2026*
