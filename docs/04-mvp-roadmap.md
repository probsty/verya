# Roadmap MVP — mini ERP Suisse (Greder Montagen)

**Produit :** ERP Montage / Chantiers  
**Date :** 7 juin 2026  
**Stack :** Angular · NestJS · PostgreSQL · Docker Compose · Docker images

---

## Note sur l'ordre de construction

> **Cet ordre est un ordre d'apprentissage et d'implémentation progressive avec Claude Code.**
>
> Il permet de construire et valider chaque entité métier indépendamment, sans la friction de l'authentification en phase initiale. Ce n'est pas l'ordre naturel d'une architecture enterprise complète — dans un projet d'équipe classique, l'auth et la sécurité seraient mises en place dès le début.
>
> L'avantage de cette approche : chaque phase produit quelque chose de tangible et testable immédiatement, sans dépendances bloquantes entre modules.

---

## Phase 0 — Documentation & Architecture

**Objectif :** Figer les décisions, documenter le domaine, préparer le terrain.

- [x] `docs/product-inventory.md` — inventaire fonctionnel & technique
- [ ] `docs/00-mvp-decisions.md` — décisions validées
- [ ] `docs/01-product-scope.md` — scope produit
- [ ] `docs/02-domain-model.md` — modèle de domaine
- [ ] `docs/03-permissions.md` — modèle de permissions
- [ ] `docs/04-mvp-roadmap.md` — cette roadmap
- [ ] `docs/05-technical-risks.md` — risques techniques
- [ ] `docs/06-open-questions.md` — questions ouvertes
- [ ] `CLAUDE.md` — guide pour Claude Code (conventions, commandes, structure)
- [ ] `docs/adr/` — Architecture Decision Records

**Livrable :** documentation complète, décisions figées, pas de code.

---

## Phase 1 — Structure du projet

**Objectif :** Créer le squelette du monorepo et les applications vides.

- Monorepo avec **pnpm workspaces**
- Application **Angular** vide (`apps/frontend`)
- Application **NestJS** vide (`apps/backend`)
- **Docker Compose** avec PostgreSQL (pgAdmin optionnel — non requis pour démarrer)
- Configuration de base : TypeScript strict, ESLint, Prettier

**Livrable :** `docker compose up` démarre PostgreSQL ; Angular et NestJS démarrent sans erreur.

---

## Phase 2 — Fondation données (Prisma)

**Objectif :** Modèle de données initial en base, migrations prêtes.

- **Prisma** setup dans NestJS
- Première migration avec les entités de référence :
  - `Organization`, `User`, `Employee`
  - `Client`, `Contact`, `ClientTariff`
  - `Project`, `ProjectMember`
  - `WorkType`, `HardwareCategory`, `HardwareItem`
  - `Vehicle`, `VehicleRate`
  - `CompanyInfo`
- Script de seeding : créer le premier tenant Greder Montagen + un Owner

**Livrable :** `prisma migrate dev` applique le schéma ; `prisma db seed` crée les données initiales.

---

## Phase 3 — Entités core (sans auth)

**Objectif :** CRUD complet des entités principales. Pas d'auth, accès libre en développement.

### 3a — Clients
- CRUD `Client` avec `Contact[]` et `ClientTariff`
- Gestion des véhicules autorisés par client
- API NestJS + interface Angular (liste tableur + formulaire modal)

### 3b — Employés
- CRUD `Employee` avec distinction interne / intérimaire
- Infos agence d'intérim (taux, contact, e-mail)
- API NestJS + interface Angular

### 3c — Projets (Chantiers)
- CRUD `Project` (création, modification, assignation équipe)
- `ProjectMember` avec rôle `chef` / `monteur`
- Filtrage par statut (actif / clôturé)
- API NestJS + interface Angular

### 3d — Pointage
- Saisie `WorkEntry` + `TimeRange` (multi-employés)
- Logique de détection des trous horaires (backend)
- Alerte UI + flag `gap_acknowledged`
- Saisie km et repas

**Livrable :** CRUD fonctionnel et testable manuellement pour chaque entité.

---

## Phase 4 — Données terrain & Facturation

**Objectif :** Compléter les données terrain et générer les dossiers de facturation.

### 4a — Données terrain complémentaires
- Quincaillerie : saisie avec multiplicateur ×N, état de staging, ajout à l'unité
- Hôtel : saisie nom / adresse / prix / nuitées
- Dépenses : saisie montant + upload fichier (photo / justificatif)
- Aléas : description + durée + photos

### 4b — Clôture du chantier
- Pop-up de confirmation de clôture
- Transition `Project.status : active → closed`
- Création automatique de l'`InvoiceDossier` (statut : `draft`)

### 4c — Moteur de facturation
- Calcul des `InvoiceLine` à partir des données terrain :
  - Heures × tarif horaire client (par type de travail)
  - Km × prix/km (par véhicule, par client)
  - Repas midi / soir (forfaits client)
  - Hôtel (prix × nuitées)
  - Quincaillerie (prix snapshot × quantité × multiplicateur)
  - Intérim (heures × taux agence, document séparé)
- Snapshot des prix au moment de la génération
- Lignes modifiables par l'Owner avant validation

### 4d — Génération PDF
- Génération serveur des 4 documents :
  - Regie-Rapport
  - Verbrauchsmaterial
  - Facture client
  - Facture intérimaire (si applicable)
- Bibliothèque à décider selon le gabarit disponible (PDFKit vs Puppeteer — voir `docs/05-technical-risks.md` R6 et `docs/06-open-questions.md` Q10)
- Arborescence des dossiers : client → chantier → documents

**Livrable :** dossier de facturation complet généré et téléchargeable en PDF.

---

## Phase 5 — Auth & permissions

**Objectif :** Sécuriser l'application avec JWT et les Guards NestJS.

- Login / logout + refresh token (JWT)
- Hachage des mots de passe (bcrypt)
- Guards Owner et Chef monteur contextuel (voir `docs/03-permissions.md`)
- Middleware d'injection de l'`organization_id` depuis le token JWT
- Filtrage des vues Angular selon le rôle
- Tests des scénarios d'autorisation (Owner vs Chef vs accès interdit)

**Livrable :** application complètement sécurisée, accès par rôle fonctionnel, prête pour la production.

---

## Post-MVP

| Fonctionnalité | Notes |
|---|---|
| Intégration Bexio | Dépend de la décision export vs sync (Q1, `docs/06-open-questions.md`) |
| Envoi e-mail des factures | SendGrid / Postmark |
| Dashboard analytique | Marges, rentabilité par chantier |
| Mode hors-ligne / PWA | Architecture dédiée, IndexedDB + sync queue |
| Compte utilisateur Monteur | `User` lié à `Employee` déjà prévu dans le modèle |
| Troisième langue (IT — Tessin) | Délai à préciser — impacte le choix lib i18n |
| Administration SaaS multi-tenant | Interface de gestion des tenants |
| PostgreSQL Row Level Security | Option future sans changement de modèle |

---

*Greder Montagen — mini ERP Suisse · Roadmap MVP · 7 juin 2026*
