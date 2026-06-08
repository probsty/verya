# Inventaire fonctionnel & technique — mini ERP Suisse

**Produit :** ERP Montage / Chantiers (Greder Montagen)
**Domaine :** Montage de meubles / agencement — Suisse
**Date :** 7 juin 2026
**État :** Prototype interactif

---

## 1. Vue d'ensemble

Outil de gestion pour une entreprise de montage opérant en Suisse. Le flux métier central :

> **Client → Chantier → Pointage & frais → Clôture → Dossier de facturation**

L'application est multilingue (FR / DE), à thème sombre/clair, et pensée mobile-first pour la saisie terrain par les chefs monteurs.

### Objectif principal
- Centraliser clients, équipe, chantiers
- Saisir heures, frais & consommables sur le terrain
- Générer automatiquement les dossiers de facturation
- Produire les documents officiels (Regie-Rapport, Verbrauchsmaterial)

### Rôles utilisateurs
- **Business Owner** — accès total (Colin)
- **Chef monteur** — défini *par chantier*, gère l'administratif du/des chantier(s) concerné(s)
- **Monteur** — exécutant, pas d'accès admin

> **Principe de rôle clé :** un même employé peut être chef monteur sur le chantier A et simple monteur sur le chantier B. Le rôle n'est pas global mais contextuel au chantier. À la connexion, un utilisateur non-owner ne voit que les chantiers où il est chef monteur.

### Stack actuelle du prototype
`React 18 (UMD)` · `Babel standalone (JSX)` · `CSS-in-JS inline + variables CSS` · `localStorage` · `i18n maison (FR/DE)` · `DM Sans` · `Impression navigateur → PDF`

> **Limite structurante :** aucune base de données réelle ni backend. Toutes les données vivent dans le *localStorage* du navigateur (mono-poste, non partagé, non sécurisé). C'est le premier chantier technique pour passer en production.

---

## 2. Écrans & navigation

9 zones principales accessibles via la sidebar (filtrée selon le rôle). Navigation secondaire par fil d'Ariane dans les dossiers de facturation et les détails de chantier.

| Écran | Contenu & fonction | Accès |
|---|---|---|
| **Connexion** | Sélection d'utilisateur, mot de passe, choix langue. Visuel plein écran 50/50 (branding + formulaire). | Public |
| **Tableau de bord** | Synthèse : chantiers actifs, à clôturer, raccourcis. Pas de chiffres financiers exposés. | Tous |
| **Clients** | Liste type tableur (tri, recherche). Fiche : société, contacts (prénom+nom), tarifs, véhicules autorisés, e-mail facturation. | Owner |
| **Équipe** | Liste des employés internes & intérimaires. Infos agence d'intérim (taux, responsable, e-mail). | Owner |
| **Chantiers** | Liste filtrable (par client, statut). Détail = cœur de l'app : équipe, pointage, frais, quincaillerie, hôtel, dépenses, aléas, clôture. | Owner + chef |
| **Pointage** | Saisie multi-employés des plages horaires par type de travail. Détection des trous horaires. | Owner + chef |
| **Factures** | Arborescence : dossier *client* → dossiers *chantiers* → documents. Tri alpha / année. Filtres client & année. | Owner |
| **Administration** | Catalogues configurables : véhicules, types de travail, types de quincaillerie, articles, infos société (signature, adresse). | Owner |
| **Profil / Réglages** | E-mail, mot de passe, langue de l'app, thème clair/sombre. | Tous |

> **Filtrage par rôle :** le chef monteur ne voit ni Clients, ni Équipe, ni Administration, ni Factures — uniquement les chantiers dont il a la charge. Le monteur a un accès encore plus restreint.

---

## 3. Composants réutilisables

Bibliothèque d'UI partagée (*erp-shared*) garantissant la cohérence visuelle entre modules.

**Structure**
- Sidebar (navigation filtrée par rôle)
- PageHeader (titre + actions)
- CardTitle (en-tête section icône + couleur)
- Card / Box, Divider, Tabs
- SectionLabel, KVRow (clé-valeur)

**Données**
- Liste type tableur (tri, colonnes)
- SearchBar (recherche instantanée)
- StatCard, Badge / Tag de statut
- EmptyState (état vide illustré)
- Filtres (catégorie, client, année)

**Saisie**
- Modal (formulaires)
- Champs : texte, nombre, select, date
- Stepper +/- (quantités quincaillerie)
- Multiplicateur ×N (meubles répétés)
- Sélection multi-employés
- Upload fichiers (photos, tickets)

**Feedback**
- Confirm (dialogue de confirmation)
- Pop-up clôture → génération factures
- Alerte trou horaire (point orange + case)
- Toggles thème & langue

---

## 4. Actions utilisateur

### Configuration (Owner)
- **Clients** — créer/modifier, ajouter contacts (prénom + nom + rôle), définir tarifs (horaire, km, repas midi/soir, hôtel), sélectionner véhicules autorisés, lignes de prix personnalisées
- **Équipe** — gérer employés internes, marquer un intérimaire et renseigner agence (taux horaire, responsable, e-mail)
- **Administration** — gérer véhicules & prix/km, types de travail (Montage, Fachzeit, Fahrzeit…), types de quincaillerie (catégories), articles, infos société & signature

### Exploitation chantier (Chef monteur)
- **Pointage** — saisir des plages horaires par type de travail pour **plusieurs employés à la fois** ; le système alerte si un trou existe entre deux plages
- **Frais saisis** — kilomètres & repas du midi renseignés directement dans le chantier
- **Quincaillerie** — rechercher/filtrer par catégorie, sélectionner des articles, appliquer un **×N** (15 meubles identiques) puis valider ; ré-ajouter ensuite des articles à l'unité
- **Hôtel** — saisir nom, adresse, prix & nombre de nuitées (souvent renseigné tard, à la clôture)
- **Dépenses & tickets** — ajouter justificatifs (parking, restaurant, hôtel) avec photo
- **Aléas** — signaler du temps perdu à cause d'un autre corps de métier (photos, description, durée) → génère un document de notification client
- **Clôture** — pop-up de confirmation proposant de générer les factures

### Facturation (Owner)
- Parcourir l'arborescence dossier client → chantier → documents (vue liste)
- Générer / consulter : **Regie-Rapport**, **Verbrauchsmaterial**, justificatifs, facture client
- Générer une facture **Regie + nom intérimaire** distincte si des intérimaires sont intervenus
- Trier par année / client, exporter en PDF via impression

---

## 5. Modèle de données

Entités principales et relations. Les montants/tarifs ne sont exposés qu'au moment de la facturation.

| Entité | Champs clés | Relations |
|---|---|---|
| **User** | username, password, role (owner/monteur), employeeId, displayName, langue, thème | → Employee |
| **Employee** | prénom, nom, adresse, e-mail, tél, taux horaire, statut intérim + infos agence | ↔ Chantiers |
| **Client** | société, adresse, e-mails (dont facturation), contacts[], tarifs (horaire/km/repas/hôtel), véhicules autorisés[], lignes custom[] | ← Chantiers |
| **Contact** | prénom, nom, rôle, e-mail, tél, mobile | ⊂ Client |
| **Chantier** | nom, client, adresse, dates, chef monteur, équipe[], véhicules, hôtel, statut, frais, aléas[] | → Client, Employés |
| **Pointage** | date, employé, plages horaires par type de travail, km, repas | ⊂ Chantier |
| **Type de travail** | code, libellé FR/DE, facturable, requiert km, couleur | config admin |
| **Quincaillerie** | réf, nom, unité, prix unitaire, catégorie | config admin |
| **Véhicule** | type (T1/T2/T3), libellé, description, prix/km (par client) | config admin |
| **Dépense / Ticket** | type, montant, justificatif (photo) | ⊂ Chantier |
| **Aléa** | description, durée perdue, photos, date | ⊂ Chantier |
| **Facture / Document** | type (Regie/Verbrauch/client/intérim), chantier, client, lignes, montants, année | → Chantier, Client |

---

## 6. États d'interface

**Vide**
- Aucun chantier / client / article
- Recherche sans résultat
- Dossier de facturation non généré

**Transitoire**
- Chantier actif vs. clôturé
- Facture brouillon vs. validée
- Saisie en cours (staging quincaillerie)

**Alerte / validation**
- Trou horaire détecté (point orange + case « j'ai conscience »)
- Confirmation de clôture / suppression
- Champs requis manquants

**Préférences**
- Thème clair / sombre
- Langue FR / DE
- Vue filtrée selon le rôle
- Responsive mobile / tablette / desktop

---

## 7. Dépendances backend probables

Pour passer du prototype à un produit multi-utilisateur en production.

| Brique | Rôle | Pistes |
|---|---|---|
| **Base de données** | Persistance relationnelle (clients, chantiers, pointages, factures) | PostgreSQL |
| **API / Backend** | Logique métier, calculs de facturation, génération documents | Node/NestJS, Django |
| **Authentification** | Login sécurisé, hachage mots de passe, sessions/JWT | Auth0 / Supabase Auth |
| **Autorisation** | RBAC contextuel (rôle par chantier) | CASL / policies |
| **Stockage fichiers** | Photos chantier, tickets, justificatifs | S3 / Supabase Storage |
| **Génération PDF** | Regie-Rapport, Verbrauchsmaterial, factures (côté serveur) | Puppeteer / wkhtmltopdf |
| **E-mail** | Envoi factures au client / e-mail facturation | SendGrid / Postmark |
| **Internationalisation** | FR / DE (et extension IT possible) | i18next |
| **Intégration compta** | Synchronisation Bexio (export factures) | API Bexio |
| **Infra / Déploiement** | Conteneurisation, CI/CD, sauvegardes | Docker, hébergeur CH |

> **Souveraineté des données :** données clients suisses → privilégier un hébergement en Suisse/UE et un chiffrement au repos (conformité nLPD).

---

## 8. Périmètre MVP

### Dans le MVP
- ✅ **Auth & rôles** — login sécurisé, owner / chef monteur (par chantier) / monteur
- ✅ **Gestion clients & équipe** — CRUD complet avec tarifs et intérim
- ✅ **Cycle de vie chantier** — création, équipe, pointage multi-employés, frais, quincaillerie, clôture
- ✅ **Génération du dossier de facturation** — Regie-Rapport + Verbrauchsmaterial + facture client + intérim
- ✅ **Export PDF & persistance serveur** — base de données réelle, fichiers stockés
- ✅ **Bilingue FR/DE & responsive mobile** — saisie terrain

### Post-MVP
- ⬜ **Intégration Bexio** — synchronisation comptable automatique
- ⬜ **Envoi e-mail intégré** — factures directement au client
- ⬜ **Tableau de bord analytique** — marges, rentabilité par chantier
- ⬜ **App mobile native / mode hors-ligne** — saisie sans réseau sur chantier
- ⬜ **Troisième langue (IT)** — couverture Tessin

---

## 9. Risques

| Sévérité | Risque | Détail |
|---|---|---|
| 🔴 Élevé | **Absence de backend / données en localStorage** | Données mono-poste, non partagées entre owner et chefs monteurs, perdues si le navigateur est vidé. Bloquant pour un usage réel multi-utilisateur. |
| 🔴 Élevé | **Sécurité & conformité nLPD** | Mots de passe en clair, aucune isolation des données. Données personnelles de clients suisses à protéger. |
| 🟠 Moyen | **Exactitude de la facturation** | La génération automatique (heures × tarif, km, repas, hôtel, intérim) doit être auditable et corrigeable manuellement avant envoi. |
| 🟠 Moyen | **Conformité des documents officiels** | Regie-Rapport & Verbrauchsmaterial doivent correspondre exactement aux gabarits de l'entreprise et aux attentes du client (Bexio). |
| 🟠 Moyen | **Modèle de rôle contextuel** | Le rôle « chef monteur par chantier » complexifie l'autorisation : un bug d'accès exposerait des données financières aux mauvaises personnes. |
| 🔵 Faible | **Adoption terrain** | La saisie mobile doit rester ultra-simple, sinon les chefs monteurs ne pointeront pas en temps réel. |
| 🔵 Faible | **Dérive du périmètre** | Beaucoup de demandes fines d'UI ; nécessite une discipline produit pour stabiliser avant industrialisation. |

---

## 10. Questions ouvertes

1. Combien d'utilisateurs simultanés à terme (chefs monteurs + owner) ? Mono-entreprise ou multi-tenant ?
2. L'intégration **Bexio** est-elle une exportation ponctuelle (PDF/CSV) ou une synchronisation API bidirectionnelle ?
3. Les tarifs (horaire, km, repas, hôtel) sont-ils figés par contrat client ou négociés chantier par chantier ?
4. Faut-il un workflow de validation owner avant qu'une facture générée par un chef ne parte au client ?
5. Le mode **hors-ligne** sur chantier (zone sans réseau) est-il indispensable au MVP ?
6. Quelle politique de conservation/archivage des dossiers de facturation (obligation légale CH = 10 ans) ?
7. Les intérimaires ont-ils un jour besoin d'un accès à l'app, ou restent-ils gérés uniquement par leur agence ?
8. Faut-il une signature électronique (chef monteur / client) sur le Regie-Rapport ?
9. Une troisième langue (IT) pour le Tessin est-elle prévue à court terme ?

---

*Greder Montagen — mini ERP Suisse · Inventaire fonctionnel & technique · 7 juin 2026*

