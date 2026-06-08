# CLAUDE.md — Veyra / Greder Montagen ERP

Règles permanentes pour Claude Code. À lire avant toute tâche.

---

## 1. Project overview

- **Produit :** ERP terrain / chantiers / pointage / facturation
- **Stack cible :** Angular, NestJS, PostgreSQL, Docker Compose, Docker images
- **Architecture cible :** monorepo (pnpm workspaces)
- `apps/web` — Frontend Angular
- `apps/api` — Backend NestJS
- `docs` — Documentation
- `infra` — Infrastructure
- `packages/shared-contracts` — Contrats partagés si nécessaire
- **Multi-tenant prévu dès le départ** avec `Organization` comme entité racine
- Greder Montagen est le premier tenant

---

## 2. Mandatory workflow

Claude doit toujours suivre ce cycle :

1. Explore
2. Plan
3. Wait for approval
4. Implement small scope
5. Test
6. Summarize
7. Mention risks

Règles :
- Toujours lire `CLAUDE.md` et les docs pertinentes avant une tâche
- Toujours proposer un plan avant modification
- Toujours attendre la validation avant de créer ou modifier des fichiers
- Ne jamais coder directement sans plan
- Ne jamais faire de commit automatiquement
- L'utilisateur relit le diff et commit lui-même

---

## 3. Scope control

- Ne jamais générer toute l'application d'un coup
- Ne jamais modifier des fichiers hors scope sans le signaler
- Ne jamais mélanger plusieurs vertical slices
- Ne jamais ajouter une dépendance sans justification
- Ne jamais faire de refactoring opportuniste non demandé
- Toujours distinguer ce qui est MVP de ce qui est post-MVP
- Si une tâche devient trop large, proposer un découpage
- Si le besoin est ambigu, lister les hypothèses avant d'implémenter

---

## 4. Tooling and generators

Claude doit utiliser les générateurs officiels quand ils sont pertinents :

- **Angular CLI** pour Angular
- **NestJS CLI** pour NestJS

Règles :
- Avant d'utiliser un générateur, expliquer la commande proposée et son impact
- Ne pas créer une app Angular manuellement si Angular CLI peut le faire proprement
- Ne pas créer une app NestJS manuellement si NestJS CLI peut le faire proprement
- Ne pas lancer de commande destructive sans validation
- Ne pas installer de dépendance sans expliquer pourquoi
- Ne pas choisir une librairie structurante sans proposer au moins une alternative
- Si une décision est structurante, proposer une ADR avant de coder

---

## 5. Architecture rules

- Architecture monorepo
- Frontend Angular dans `apps/web`
- Backend NestJS dans `apps/api`
- Documentation dans `docs`
- Infrastructure dans `infra`
- Shared contracts dans `packages/shared-contracts` si nécessaire
- PostgreSQL comme base de données
- Prisma comme ORM, sauf décision contraire documentée par ADR
- L'architecture doit rester évolutive, testable et maintenable
- Les décisions structurantes doivent être documentées dans des ADR avant implémentation

---

## 6. Angular rules

- Utiliser les conventions officielles Angular et Angular CLI
- Utiliser une structure feature-based :
  - `core/`
  - `shared/`
  - `features/`
  - `layout/`
- Favoriser les composants standalone si cohérent avec la version Angular utilisée
- Utiliser lazy loading pour les grandes features
- Le frontend aide l'utilisateur, mais ne remplace jamais les validations backend
- Ne pas hardcoder l'URL API dans les composants
- Prévoir loading, empty et error states
- Garder l'UX mobile-first pour la saisie terrain
- Garder l'i18n FR/DE en tête dès la structure, sans sur-implémenter trop tôt
- Ne pas créer de design system générique trop tôt
- Extraire des composants partagés uniquement lorsqu'un pattern est répété et stabilisé

---

## 7. NestJS / Backend architecture rules

Le backend doit être conçu avec Domain-Driven Design et architecture hexagonale.

Principes obligatoires :

- Le domaine ne doit pas dépendre de NestJS
- Le domaine ne doit pas dépendre de Prisma
- Le domaine ne doit pas dépendre d'un framework externe
- Les règles métier doivent vivre dans le domaine ou dans les use cases applicatifs
- Les controllers NestJS sont des adapters entrants
- Prisma est un adapter sortant
- Les repositories doivent être définis comme ports/interfaces côté application ou domaine
- Les implémentations Prisma doivent rester dans l'infrastructure
- Les DTO HTTP ne doivent pas devenir des modèles de domaine
- Les entités de domaine ne doivent pas être les entités Prisma
- Les calculs métier ne doivent pas être dans les controllers
- Les transactions doivent être gérées au niveau application/service adapté, pas dans les controllers

---

## 8. Backend folder structure

Structure cible pour `apps/api/src/` :

```
apps/api/src/
├── api/
│   ├── http/
│   │   ├── filters/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── decorators/
│   │   └── presenters/
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
│   │   │   ├── handlers/
│   │   │   ├── managers/
│   │   │   ├── ports/
│   │   │   ├── commands/
│   │   │   └── queries/
│   │   ├── infrastructure/
│   │   │   ├── persistence/
│   │   │   ├── prisma/
│   │   │   └── mappers/
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
    ├── domain/
    ├── application/
    ├── infrastructure/
    ├── presentation/
    └── utils/
```

Définitions :

- `api/` — éléments transverses HTTP : filters, guards, interceptors, decorators, presenters
- `features/` — bounded contexts métier : clients, employees, projects, time-tracking, invoicing, identity-access
- `domain/` — métier pur : entities, value objects, domain services, domain events
- `application/` — orchestration applicative : handlers, managers, ports, commands, queries
- `infrastructure/` — adapters sortants : Prisma repositories, file storage, PDF generation, external services, mappers
- `presentation/` — adapters entrants propres à la feature : HTTP controllers, HTTP DTOs
- `shared/` — éléments réellement réutilisables entre plusieurs contexts uniquement

---

## 9. Handlers and managers

**Handlers** — un handler exécute un cas d'usage précis :

- `CreateClientHandler`
- `UpdateClientHandler`
- `AssignProjectTeamHandler`
- `AddTimeEntryHandler`
- `GenerateInvoiceDraftHandler`

**Managers** — un manager orchestre un processus métier plus large impliquant plusieurs handlers, repositories ou domain services :

- `ProjectClosingManager`
- `InvoiceGenerationManager`
- `TimeEntryValidationManager`

Règles :
- Controllers call handlers or managers
- Controllers must stay thin
- Controllers do not contain business logic
- Prisma stays in infrastructure
- Domain does not depend on Prisma, NestJS or HTTP DTOs
- DTOs do not leak into domain entities
- Repositories are defined as ports and implemented as infrastructure adapters
- Les handlers doivent être testables sans HTTP
- Les managers ne doivent pas devenir des services fourre-tout
- Ne pas créer de manager sans orchestration réelle

---

## 10. Multi-tenant rules

- `Organization` est l'entité racine
- Les données métier principales doivent être rattachées à `Organization`
- Toute query backend doit respecter l'isolation par `organizationId`
- Ne jamais permettre de fuite cross-tenant
- Les tests doivent couvrir l'isolation multi-tenant
- Greder Montagen est le premier tenant, mais le modèle doit rester extensible
- Ne pas construire une plateforme SaaS complète dans le MVP :
  - pas de billing SaaS
  - pas d'onboarding multi-tenant avancé
  - pas de super-admin complexe sauf décision ADR future

Entités principales à rattacher à `Organization` :
`User` · `Employee` · `Client` · `Project` · `Invoice / InvoiceDossier` · `WorkType` · `HardwareItem` · `Vehicle` · `CompanyInfo`

---

## 11. Backend implementation rules

- Utiliser des modules NestJS context-based ou feature-based
- Ne pas exposer directement les entités Prisma
- Utiliser DTOs pour input/output HTTP
- Utiliser des mappers entre Prisma, domain et response DTOs
- Les règles métier critiques doivent être testées
- Les calculs de facturation doivent être faits côté backend
- Les montants doivent éviter les floats — utiliser `Decimal`, integer cents, ou une stratégie monétaire validée par ADR
- Les erreurs métier doivent être explicites
- Les cas d'usage doivent être testables sans HTTP
- Les use cases ne doivent pas dépendre directement des controllers
- Les transactions ne doivent pas être pilotées depuis les controllers

---

## 12. Security rules

- Ne jamais hardcoder de secrets
- Ne jamais stocker de mot de passe en clair
- Ne jamais stocker les fichiers uploadés en base64
- Ne jamais exposer de chemins filesystem
- Les permissions finales sont validées côté backend
- Le frontend peut masquer des boutons, mais ne doit jamais être la seule barrière de sécurité
- Les données personnelles ne doivent pas être loggées inutilement
- Préserver les contraintes nLPD / données suisses

---

## 13. Business rules

- Le Business Owner valide toujours les factures finales
- Le Chef monteur clôture les chantiers, mais ne valide pas la facture finale
- Le Chef monteur est un rôle contextuel par chantier (stocké dans `ProjectMember.role`)
- Dans le MVP, les monteurs n'ont pas de compte utilisateur
- Les factures doivent snapshotter les lignes, quantités, prix, totaux, TVA, arrondis et devise
- Le mode hors-ligne est hors MVP
- Une facture validée ne doit pas être recalculée depuis les tarifs courants
- Les données de facturation doivent être auditables

---

## 14. Testing rules

- Ajouter des tests sur les règles métier critiques
- Tester les permissions contextuelles
- Tester les calculs horaires
- Tester les calculs de facturation
- Tester l'isolation multi-tenant
- Tester les use cases backend sans passer obligatoirement par HTTP
- Tester les mappers critiques
- Ne pas se contenter de tests de controllers pour les règles métier

---

## 15. Git rules

- Ne pas faire de commit automatiquement
- Résumer les fichiers modifiés
- L'utilisateur relit le diff et commit lui-même
- Après chaque tâche, indiquer :
  - fichiers créés
  - fichiers modifiés
  - tests lancés
  - tests non lancés
  - risques restants
  - prochaine étape recommandée

---

*Greder Montagen — mini ERP Suisse · CLAUDE.md · Juin 2026*
