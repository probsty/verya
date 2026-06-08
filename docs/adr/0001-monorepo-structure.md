# ADR 0001 — Monorepo Structure

## Status

Accepted

## Context

Le projet Veyra / Greder Montagen comprend un frontend Angular, un backend NestJS, une infrastructure Docker, et potentiellement des contrats partagés entre les deux applications (types, interfaces). Sans structure commune, les deux applications risquent de diverger dans leurs conventions, de dupliquer des types, et de compliquer la navigation dans le code.

L'équipe est petite (un seul développeur dans la phase initiale). La cohérence et la simplicité de navigation priment sur la complexité d'un système de build distribué.

## Decision

Le projet est organisé en **monorepo pnpm workspaces** avec la structure suivante :

```
/
├── apps/
│   ├── web/          # Frontend Angular
│   └── api/          # Backend NestJS
├── packages/
│   └── shared-contracts/   # Types et interfaces partagés (si nécessaire)
├── docs/             # Documentation projet et ADR
├── infra/            # Configuration Docker, scripts de déploiement
├── CLAUDE.md         # Règles permanentes pour Claude Code
└── package.json      # Workspace root (pnpm)
```

`packages/shared-contracts` ne sera créé que si un besoin de partage de types entre `apps/web` et `apps/api` est avéré. Il n'est pas créé par défaut.

## Alternatives considered

**Nx** — système de monorepo avancé avec cache de build, générateurs et graphe de dépendances. Écarté car sur-dimensionné pour un projet de cette taille et ajoute une couche d'abstraction supplémentaire à maîtriser.

**Turborepo** — pipeline de build distribué. Même raison que Nx : complexité injustifiée pour la phase initiale. Peut être adopté plus tard si les temps de build deviennent un problème.

**Repos séparés (frontend + backend)** — simplifie les permissions git mais complique la navigation, la synchronisation des types et les changements cross-app. Écarté car le gain est faible pour une équipe solo.

## Consequences

- **Positif :** navigation unifiée dans un seul repo, commandes partagées (`pnpm install` à la racine), facilité pour les changements impactant frontend et backend simultanément.
- **Positif :** `shared-contracts` permet de partager des types DTO sans duplication si le besoin émerge.
- **Négatif :** pnpm workspaces nécessite une configuration initiale rigoureuse des scopes de packages.
- **Vigilance :** ne pas créer de couplage fort via `shared-contracts` trop tôt. Les contrats partagés doivent rester des types légers (DTOs, enums), jamais de logique métier.
- **Vigilance :** les scripts de CI/CD doivent être configurés pour ne rebuilder que les apps impactées par un changement.

---

*Veyra — Greder Montagen · ADR 0001 · Juin 2026*
