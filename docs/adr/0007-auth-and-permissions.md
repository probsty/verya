# ADR 0007 — Authentication and Permissions

## Status

Accepted

## Context

L'application doit gérer deux types d'accès distincts : un accès global à l'organisation (Business Owner) et un accès contextuel par projet (Chef monteur). Une erreur dans le modèle de permissions permettrait à un Chef monteur de consulter les données financières d'un client, ou à un utilisateur d'un tenant de lire les données d'un autre.

Par choix délibéré de la roadmap (voir `docs/04-mvp-roadmap.md`), l'authentification est implémentée en Phase 5 — intentionnellement en dernier pour permettre d'itérer rapidement sur les entités métier pendant les phases 2 à 4 sans friction d'auth. Le modèle de données doit néanmoins être prévu dès la Phase 2.

## Decision

### Modèle de rôles

```
User.role = 'owner'            → Business Owner : accès global à l'Organization
User.role = 'member'           → Utilisateur standard : accès selon ses ProjectMember
ProjectMember.role = 'chef'    → Chef monteur sur CE projet spécifique
ProjectMember.role = 'monteur' → Monteur sur CE projet (sans accès app dans le MVP)
```

**Business Owner** (`User.role = 'owner'`) :
- Rôle global à l'`Organization`
- Accès complet à toutes les données et actions
- Seul rôle pouvant valider les factures

**Chef monteur** (`ProjectMember.role = 'chef'`) :
- Rôle contextuel par `Project` — stocké dans `ProjectMember`, jamais dans `User`
- Un même utilisateur peut être Chef sur le Projet A et non-chef sur le Projet B
- Voit uniquement les projets où il est Chef monteur
- Peut clôturer un chantier mais ne valide pas la facture finale

**Monteur** (`ProjectMember.role = 'monteur'`) :
- Hors MVP — pas de compte `User`
- Existe comme entité `Employee` assignée à des projets
- N'accède pas à l'application dans le MVP

### Mécanisme d'authentification

- **JWT** (JSON Web Token) avec claims `userId`, `organizationId`, `role`
- Refresh token pour les sessions longues (terrain)
- Mots de passe hachés avec **bcrypt** (coût ≥ 12)
- L'`organizationId` et l'`employeeId` sont toujours lus depuis le token JWT, jamais depuis le payload HTTP

### Guards NestJS (Phase 5)

| Guard | Vérification |
|---|---|
| `AuthGuard` | Token JWT valide + `organization_id` du token correspond à la ressource |
| `OwnerGuard` | `User.role === 'owner'` |
| `ProjectChefGuard` | `ProjectMember WHERE project_id = :id AND employee_id = :empId AND role = 'chef'` |
| `OwnerOrChefGuard` | Owner OU ProjectChef (pour les actions terrain communes) |

### Ordre d'implémentation

Le modèle de données (`User`, `Employee`, `ProjectMember` avec leurs relations) est créé en **Phase 2** avec le schéma Prisma initial.

L'implémentation technique de l'auth (JWT, guards, middleware d'injection de l'`organizationId`) est réalisée en **Phase 5**, après que toutes les entités métier sont fonctionnelles. En phases 2 à 4, l'accès est libre en développement.

## Alternatives considered

**Auth0 / Supabase Auth** — gestion de l'authentification déléguée à un service tiers. Écarté car ajoute une dépendance externe, complexifie les tests, et le RBAC contextuel par projet (Chef monteur) nécessite de toute façon une logique custom côté backend.

**Passport.js seul** — bibliothèque d'authentification bas niveau. Peut être utilisé avec NestJS via `@nestjs/passport` pour les stratégies JWT. Décision de librairie précise à confirmer en Phase 5.

**CASL** — bibliothèque d'autorisation déclarative. Intéressante pour des règles complexes. Peut être évaluée en Phase 5 si les guards NestJS natifs s'avèrent insuffisants.

**Rôle Chef monteur global sur User** — plus simple mais incorrect métier : un employé peut être Chef sur un chantier et non-chef sur un autre. Écarté par conception.

## Consequences

- **Positif :** le rôle contextuel par projet reflète fidèlement la réalité métier.
- **Positif :** l'implémentation en Phase 5 permet d'itérer rapidement sur les entités métier en phases 2-4.
- **Positif :** le modèle de données prévu dès la Phase 2 évite un refactoring de schéma en Phase 5.
- **Négatif :** pendant les phases 2-4, l'application est non sécurisée — ne pas déployer sur un environnement accessible en dehors du développement local.
- **Vigilance :** risque identifié dans `docs/05-technical-risks.md` — R5. Tests d'autorisation obligatoires : chaque route doit avoir un test "accès interdit" pour les rôles non autorisés.
- **Vigilance :** le frontend peut masquer des boutons selon le rôle, mais les permissions finales sont toujours validées côté backend.

---

*Veyra — Greder Montagen · ADR 0007 · Juin 2026*
