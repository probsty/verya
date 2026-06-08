# Décisions MVP — mini ERP Suisse (Greder Montagen)

**Date de validation :** 7 juin 2026  
**Produit :** ERP Montage / Chantiers  
**Source :** Décisions validées lors de la phase de documentation initiale

---

Ce document est la source of truth pour les décisions architecturales et produit prises avant le démarrage de l'implémentation. En cas de doute lors du développement, ce fichier fait référence.

---

## Décision 1 — Architecture multi-tenant

**Statut :** ✅ Validé

### Décision
L'application est conçue pour servir plusieurs entreprises (tenants) dès le départ.

- Entité `Organization` créée dès le premier schéma de données
- Toutes les données métier sont rattachées à une `Organization` : `User`, `Employee`, `Client`, `Project`, `InvoiceDossier`, `WorkType`, `HardwareItem`, `Vehicle`, `CompanyInfo`
- L'isolation des données entre tenants est garantie au niveau applicatif (filtre `WHERE organization_id = ?` systématique)
- Greder Montagen sera le premier tenant, créé par script de seeding

### Ce qui n'est PAS dans le MVP
- Administration SaaS (gestion des tenants depuis une interface)
- Billing SaaS / abonnements
- Onboarding multi-entreprise automatisé
- Interface de création d'entreprise en libre-service

### Pourquoi ce choix
Réécrire le modèle de données après le MVP pour ajouter le multi-tenant serait coûteux. L'ajout de `organization_id` dès le départ est un surcoût minimal qui évite une migration douloureuse.

---

## Décision 2 — Factures comme snapshots immuables

**Statut :** ✅ Validé

### Décision
Une facture générée est un snapshot complet et immuable au moment de sa création.

Chaque ligne de facture (`InvoiceLine`) stocke :
- Description
- Quantité
- Prix unitaire au moment de la génération
- Total calculé
- Taux de TVA appliqué
- Devise (CHF par défaut)

**Un changement de tarif client après génération n'affecte jamais les factures déjà créées.**

### Ce que cela implique dans le code
- Pas de `JOIN` dynamique entre `InvoiceLine` et `ClientTariff` pour recalculer
- Les lignes de facture sont des données dénormalisées et indépendantes des référentiels tarifaires courants
- Correction possible avant validation : l'Owner peut modifier les lignes en statut `draft`

---

## Décision 3 — Workflow de validation des factures

**Statut :** ✅ Validé

### Décision
```
Chef monteur saisit les données terrain
        ↓
Chef monteur clôture le chantier
        ↓
Système génère l'InvoiceDossier (statut : draft)
        ↓
Business Owner contrôle et corrige si nécessaire
        ↓
Business Owner valide → statut : validated
```

Le **Chef monteur ne valide pas la facture finale**. La validation est une action exclusive du Business Owner.

### États de l'InvoiceDossier
- `draft` — généré à la clôture, modifiable par l'Owner
- `validated` — validé par l'Owner, immuable

---

## Décision 4 — Rôle Monteur dans le MVP

**Statut :** ✅ Validé

### Décision
Dans le MVP, **les monteurs n'ont pas de compte utilisateur**.

- Ils existent comme entités `Employee` assignées à des projets
- Ils apparaissent dans les pointages (saisis par le Chef monteur)
- Ils n'accèdent pas à l'application

**Les deux seuls rôles avec compte utilisateur dans le MVP sont :**
1. **Business Owner** — accès global à l'Organization
2. **Chef monteur** — rôle contextuel par chantier (stocké dans `ProjectMember.role`)

Le rôle Chef monteur n'est pas global. Un même `Employee` peut être Chef sur le Chantier A et simple monteur (sans accès) sur le Chantier B.

### Post-MVP
L'ajout d'un compte Monteur est envisagé mais hors périmètre MVP. Le modèle de données prévoit cette évolution via l'entité `Employee` déjà existante.

---

## Décision 5 — Mode hors-ligne

**Statut :** ✅ Validé

### Décision
Le mode hors-ligne est **hors périmètre MVP**.

- L'application nécessite une connexion internet
- Aucun Service Worker offline
- Aucun IndexedDB pour la persistance locale
- Aucune queue de synchronisation
- Aucune résolution de conflits offline

### Post-MVP
Le mode hors-ligne (PWA avec sync) reste un besoin terrain identifié. Il sera traité dans une phase ultérieure avec une architecture dédiée (IndexedDB + sync queue + résolution de conflits).

---

*Greder Montagen — mini ERP Suisse · Décisions MVP · 7 juin 2026*
