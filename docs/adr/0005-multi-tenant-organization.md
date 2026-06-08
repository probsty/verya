# ADR 0005 — Multi-Tenant Architecture with Organization

## Status

Accepted

## Context

Greder Montagen est le premier client de l'application, mais le modèle doit pouvoir accueillir d'autres entreprises à terme. Réécrire le modèle de données après le MVP pour ajouter le multi-tenant serait coûteux et risqué (migrations de schéma complexes, refactoring de toutes les queries).

L'application gère des données commercialement sensibles (tarifs clients, factures, données personnelles). Une fuite de données entre deux tenants aurait des conséquences légales graves (nLPD).

## Decision

L'application est conçue **multi-tenant dès le premier schéma de données**, avec `Organization` comme entité racine.

### Entités directement rattachées à Organization

`User` · `Employee` · `Client` · `Project` · `InvoiceDossier` · `WorkType` · `HardwareCategory` · `HardwareItem` · `Vehicle` · `CompanyInfo`

### Règles d'isolation

**Règle 1 — Filtre systématique**
Chaque query Prisma sur une entité métier inclut `WHERE organization_id = :orgId`. Sans ce filtre, les données d'un tenant peuvent être lues par un autre.

**Règle 2 — Source du organizationId**
L'`organizationId` est toujours extrait du token JWT (claim `organization_id`), jamais depuis le payload HTTP du client.

**Règle 3 — Catalogues par tenant**
`WorkType`, `HardwareItem`, `HardwareCategory`, `Vehicle`, `CompanyInfo` sont des données propres à chaque `Organization`. Il n'existe pas de catalogue global partagé entre tenants.

**Règle 4 — Tests d'isolation obligatoires**
Chaque feature doit inclure des tests vérifiant qu'un tenant A ne peut pas lire ou modifier les données d'un tenant B.

### Initialisation du premier tenant

Greder Montagen est créé par un **script de seeding** (`prisma db seed`) lors de la mise en production initiale. La méthode exacte (script Manuel ou endpoint sécurisé) sera tranchée avant la Phase 2 (voir `docs/06-open-questions.md` — Q6).

### Ce qui n'est PAS dans le MVP

- Interface d'administration SaaS (gestion des tenants)
- Billing SaaS / abonnements
- Onboarding multi-entreprise automatisé
- Interface de création d'entreprise en libre-service
- Super-admin cross-tenant (sauf ADR future)

### PostgreSQL Row Level Security

PostgreSQL RLS peut être ajouté comme filet de sécurité supplémentaire dans une phase future, sans changement du schéma de données. Ce n'est pas une priorité MVP.

## Alternatives considered

**Schema-per-tenant** — chaque tenant a son propre schéma PostgreSQL. Isolation forte au niveau DB. Écarté car les migrations doivent être appliquées sur N schémas simultanément, et Prisma ne gère pas nativement cette configuration.

**Database-per-tenant** — isolation maximale, infrastructure par tenant. Écarté car sur-dimensionné pour la phase initiale et complexifie le déploiement.

**Mono-tenant (une seule entreprise)** — plus simple à démarrer. Écarté car la migration vers le multi-tenant après le MVP est coûteuse (toutes les entités à migrer, toutes les queries à modifier). L'ajout de `organization_id` dès le départ est un surcoût minimal.

## Consequences

- **Positif :** le modèle de données supporte plusieurs tenants sans refactoring futur.
- **Positif :** isolation des données garantie si les règles de filtrage sont respectées.
- **Positif :** PostgreSQL RLS peut être ajouté plus tard comme couche supplémentaire sans migration.
- **Négatif :** toutes les queries doivent inclure `organizationId` — un oubli crée une fuite de données.
- **Vigilance :** risque élevé identifié dans `docs/05-technical-risks.md` — R1. Mitigation : helper NestJS injectant automatiquement l'`organizationId` depuis le contexte de requête + tests d'intégration cross-tenant.
- **Vigilance :** ne pas ajouter de fonctionnalités SaaS (billing, onboarding automatisé) dans le MVP sans une ADR dédiée.

---

*Veyra — Greder Montagen · ADR 0005 · Juin 2026*
