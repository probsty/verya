# Scope produit — mini ERP Suisse (Greder Montagen)

**Produit :** ERP Montage / Chantiers  
**Entreprise :** Greder Montagen, Suisse  
**Stack :** Angular · NestJS · PostgreSQL · Docker

---

## Présentation

Greder Montagen est une entreprise de montage de meubles et d'agencement opérant en Suisse. L'application est un outil de gestion interne (mini ERP) couvrant le cycle complet d'un chantier, de la création du dossier client jusqu'à la génération des documents de facturation.

L'outil remplace un prototype basé sur localStorage (React, mono-poste, non partagé) par une application multi-utilisateur avec persistance serveur.

---

## Objectifs principaux

1. **Centraliser** les données clients, l'équipe et les chantiers
2. **Saisir sur le terrain** les heures, les frais et les consommables (mobile-first)
3. **Générer automatiquement** les dossiers de facturation à la clôture d'un chantier
4. **Produire les documents officiels** (Regie-Rapport, Verbrauchsmaterial) conformes aux attentes des clients

---

## Flux métier central

```
Client → Chantier → Pointage & frais terrain → Clôture → Dossier de facturation
```

Détail :

```
Owner crée / gère Clients (tarifs, contacts, véhicules autorisés)
         ↓
Owner crée Chantier → assigne Équipe → désigne Chef monteur
         ↓
Chef monteur sur terrain :
  ├── Pointage (plages horaires, type de travail, multi-employés)
  ├── Frais (km, repas midi/soir)
  ├── Quincaillerie (articles catalogue, ×N pour meubles répétés)
  ├── Hôtel (nom, prix, nuitées)
  ├── Dépenses + justificatifs (photos, tickets)
  └── Aléas (temps perdu, photos, notification client)
         ↓
Chef monteur → Clôture du chantier
         ↓
Système → génère InvoiceDossier (statut : draft)
         ↓
Owner → contrôle, corrige si nécessaire, valide
         ↓
Export PDF : Regie-Rapport + Verbrauchsmaterial + Facture client (+ Facture intérim si applicable)
```

---

## Contraintes UX non négociables

- **Mobile-first** : la saisie terrain par les chefs monteurs doit fonctionner sur smartphone
- **Bilingue FR/DE** : interface et documents disponibles en français et en allemand
- **Thème clair/sombre** : préférence par utilisateur
- **Responsive** : mobile / tablette / desktop

---

## Périmètre MVP

| Fonctionnalité | Incluse | Justification |
|---|---|---|
| Auth & rôles (Owner / Chef monteur contextuel) | ✅ | Accès sécurisé multi-utilisateur |
| CRUD Clients (contacts, tarifs, véhicules) | ✅ | Données de base nécessaires |
| CRUD Équipe (internes + intérimaires) | ✅ | Nécessaire pour le pointage |
| Catalogues admin (types travail, quincaillerie, véhicules) | ✅ | Référentiels métier |
| Cycle de vie chantier complet | ✅ | Flux central du produit |
| Pointage multi-employés + détection trous horaires | ✅ | Saisie terrain principale |
| Frais (km, repas), quincaillerie (×N), hôtel, dépenses | ✅ | Données pour la facturation |
| Aléas (photos, durée, notification) | ✅ | Besoin documenté du terrain |
| Clôture de chantier | ✅ | Déclencheur de la facturation |
| Génération dossier facturation (4 documents) | ✅ | Objectif principal du produit |
| Export PDF (Regie-Rapport, Verbrauchsmaterial, factures) | ✅ | Livrable final pour les clients |
| Persistance PostgreSQL + stockage fichiers | ✅ | Remplacement du localStorage |
| Architecture multi-tenant (Organization) | ✅ | Décision structurante, coût minimal maintenant |
| Bilingue FR/DE | ✅ | Contrainte non négociable |
| Responsive mobile | ✅ | Contrainte non négociable |

---

## Post-MVP

| Fonctionnalité | Justification du report |
|---|---|
| Intégration Bexio | Dépend de décisions ouvertes (export ponctuel vs sync API) |
| Envoi e-mail intégré des factures | Complexité faible mais hors flux minimal |
| Dashboard analytique (marges, rentabilité) | Valeur ajoutée, non bloquant |
| Mode hors-ligne / PWA | Architecture dédiée — décision validée comme hors-MVP |
| Compte utilisateur Monteur | Non requis pour le flux MVP |
| Troisième langue (IT — Tessin) | Délai à préciser |
| Administration SaaS multi-tenant | Multi-tenant technique prévu, admin SaaS non |
| PostgreSQL Row Level Security | Option future sans changement de modèle |

---

*Greder Montagen — mini ERP Suisse · Scope produit · 7 juin 2026*
