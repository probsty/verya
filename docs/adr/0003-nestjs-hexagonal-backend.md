# ADR 0003 — NestJS Backend with Hexagonal Architecture

## Status

Accepted

## Context

Le backend doit couvrir plusieurs bounded contexts métier (clients, employés, chantiers, pointage, facturation, identité/accès). Les règles métier sont complexes : calculs de facturation, snapshots immuables, isolation multi-tenant, permissions contextuelles par projet.

Une architecture CRUD classique (controllers → services → Prisma) concentrerait la logique métier dans des services qui dépendraient directement de Prisma et de NestJS, rendant les règles métier difficiles à tester et à faire évoluer indépendamment de l'infrastructure.

## Decision

Le backend est développé avec **NestJS** en suivant les principes du **Domain-Driven Design (DDD)** et de l'**architecture hexagonale**.

### Structure cible — `apps/api/src/`

```
apps/api/src/
├── api/
│   ├── http/
│   │   ├── filters/          # Exception filters globaux
│   │   ├── guards/           # Guards transverses (JWT, Owner, ProjectChef)
│   │   ├── interceptors/     # Logging, transformation de réponse
│   │   ├── decorators/       # @CurrentUser, @OrganizationId, etc.
│   │   └── presenters/       # Formatage des réponses HTTP
│   └── rest-api.module.ts
│
├── features/
│   ├── clients/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   ├── value-objects/
│   │   │   ├── events/
│   │   │   └── services/
│   │   ├── application/
│   │   │   ├── handlers/     # CreateClientHandler, UpdateClientHandler…
│   │   │   ├── managers/     # Orchestration multi-handlers si nécessaire
│   │   │   ├── ports/        # Interfaces Repository (IClientRepository)
│   │   │   ├── commands/
│   │   │   └── queries/
│   │   ├── infrastructure/
│   │   │   ├── prisma/       # PrismaClientRepository implements IClientRepository
│   │   │   └── mappers/      # Prisma model ↔ domain entity
│   │   ├── presentation/
│   │   │   └── http/
│   │   │       ├── controllers/
│   │   │       └── dto/
│   │   └── clients.module.ts
│   │
│   ├── employees/
│   ├── projects/
│   ├── time-tracking/
│   ├── invoicing/
│   └── identity-access/
│
└── shared/
    ├── domain/           # Value objects partagés (Money, OrganizationId…)
    ├── application/      # Interfaces et abstractions applicatives partagées
    ├── infrastructure/   # Prisma client partagé, base repository
    ├── presentation/     # DTOs partagés
    └── utils/
```

### Règles d'architecture obligatoires

**Handlers** — un handler = un cas d'usage précis :
- `CreateClientHandler`, `UpdateClientHandler`, `AssignProjectTeamHandler`
- `AddTimeEntryHandler`, `GenerateInvoiceDraftHandler`
- Testables sans HTTP, sans dépendance directe aux controllers

**Managers** — un manager orchestre un processus métier impliquant plusieurs handlers, repositories ou domain services :
- `ProjectClosingManager`, `InvoiceGenerationManager`, `TimeEntryValidationManager`
- Ne pas créer de manager sans orchestration réelle (risque de service fourre-tout)

**Séparation stricte :**
- Le domaine ne dépend pas de NestJS, Prisma ou des DTO HTTP
- Les entités de domaine ne sont pas les entités Prisma
- Les DTO HTTP ne deviennent pas des modèles de domaine
- Les repositories sont définis comme ports (interfaces) dans `application/ports/`
- Les implémentations Prisma restent dans `infrastructure/`
- Les calculs métier vivent dans le domaine ou les use cases, jamais dans les controllers
- Les transactions sont gérées au niveau application/service, jamais dans les controllers

## Alternatives considered

**NestJS CRUD classique (controllers → services → Prisma)** — plus rapide à démarrer, mais concentre la logique métier dans des services couplés à Prisma. Difficile à tester et à faire évoluer quand les règles de facturation ou les permissions deviennent complexes. Écarté.

**Express/Fastify sans framework** — plus de contrôle, moins d'opinions. Écarté car NestJS impose des conventions (DI, modules, décorateurs) qui conviennent bien à un projet d'équipe structuré avec DDD.

**Django/FastAPI (Python)** — stack différente du prototype existant, courbe d'apprentissage supplémentaire dans le contexte de ce projet. Écarté.

## Consequences

- **Positif :** les règles métier (calculs horaires, facturation, permissions contextuelles) sont testables sans démarrer HTTP ni Prisma.
- **Positif :** les implémentations Prisma peuvent être remplacées sans toucher au domaine.
- **Positif :** la structure est claire et prévisible pour chaque nouveau bounded context.
- **Négatif :** plus de fichiers et de couches qu'une architecture CRUD — verbosité accrue au démarrage.
- **Vigilance :** `shared/` ne doit contenir que du code réellement partagé entre plusieurs features. Un élément utilisé par une seule feature reste dans cette feature.
- **Vigilance :** NestJS CLI doit être utilisé pour générer les modules, controllers et providers — ne pas les créer manuellement.

---

*Veyra — Greder Montagen · ADR 0003 · Juin 2026*
