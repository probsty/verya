# Modèle de domaine — mini ERP Suisse (Greder Montagen)

**Produit :** ERP Montage / Chantiers  
**Date :** 7 juin 2026

---

## Organization comme entité racine (multi-tenant)

L'entité `Organization` est la racine de toutes les données métier. Toutes les entités principales y sont directement rattachées via un champ `organization_id`.

**Entités rattachées directement à Organization :**  
`User` · `Employee` · `Client` · `Project` · `InvoiceDossier` · `WorkType` · `HardwareCategory` · `HardwareItem` · `Vehicle` · `CompanyInfo`

Cette décision est structurante : elle garantit l'isolation des données entre tenants sans refactoring ultérieur. Voir [docs/00-mvp-decisions.md](./00-mvp-decisions.md) — Décision 1.

---

## Entités et relations

### Organization
Racine multi-tenant. Chaque entreprise cliente est un tenant.

| Champ | Type | Description |
|---|---|---|
| id | UUID | Clé primaire |
| name | string | Nom de l'entreprise (ex : "Greder Montagen") |
| slug | string | Identifiant URL-safe unique |
| settings | JSON | Préférences globales (langue par défaut, etc.) |
| created_at | timestamp | |

---

### User
Compte utilisateur. Lié à un `Employee` dans la majorité des cas.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| employee_id | UUID? | FK → Employee (nullable si compte technique) |
| email | string | Identifiant de connexion |
| password_hash | string | bcrypt |
| role | enum | `owner` \| `member` |
| display_name | string | |
| lang | enum | `fr` \| `de` |
| theme | enum | `light` \| `dark` |

> `role: member` signifie "peut être chef monteur sur certains projets". Le rôle contextuel est stocké dans `ProjectMember`, pas ici.

---

### Employee
Employé de l'organisation (interne ou intérimaire). Peut ou non avoir un compte `User`.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| first_name | string | |
| last_name | string | |
| email | string? | |
| phone | string? | |
| address | string? | |
| hourly_rate | Decimal | Taux horaire interne |
| is_interim | boolean | Intérimaire ou employé interne |
| agency_name | string? | Nom de l'agence (si intérimaire) |
| agency_contact | string? | Responsable agence |
| agency_email | string? | |
| agency_hourly_rate | Decimal? | Taux facturé par l'agence |

---

### Client
Client de Greder Montagen (donneur d'ordre d'un chantier).

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| company | string | Raison sociale |
| address | string | |
| billing_email | string | E-mail pour les factures |
| notes | string? | |

Relations : `contacts[]` (1-N Contact), `tariff` (1-1 ClientTariff), `allowed_vehicles[]` (N-N Vehicle via table de jonction)

---

### Contact
Contact rattaché à un client.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| client_id | UUID | FK → Client |
| first_name | string | |
| last_name | string | |
| role | string | Ex : "Chef de projet", "Comptable" |
| email | string? | |
| phone | string? | |
| mobile | string? | |

---

### ClientTariff
Grille tarifaire du client. Sert de base au calcul de facturation.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| client_id | UUID | FK → Client (1-1) |
| hourly_rate | Decimal | Taux horaire |
| km_rate | Decimal | Prix par kilomètre (référence, peut varier par véhicule) |
| lunch_rate | Decimal | Indemnité repas midi |
| dinner_rate | Decimal | Indemnité repas soir |
| hotel_rate | Decimal | Forfait hôtel par nuit |
| custom_lines | JSON | Lignes tarifaires personnalisées |

> Ces tarifs sont copiés en snapshot dans `InvoiceLine` à la génération. Un changement ultérieur n'affecte pas les factures existantes. Voir [docs/00-mvp-decisions.md](./00-mvp-decisions.md) — Décision 2.

---

### Project (Chantier)
Cœur de l'application. Contient toutes les données terrain.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| client_id | UUID | FK → Client |
| name | string | Nom du chantier |
| address | string | Adresse du chantier |
| start_date | date? | |
| end_date | date? | |
| status | enum | `active` \| `closed` |
| version | integer | Optimistic locking |

Relations : `members[]` (N-N Employee via ProjectMember), `work_entries[]`, `hardware[]`, `hotel?`, `expenses[]`, `incidents[]`, `invoice_dossier?`

---

### ProjectMember
Relation entre un `Project` et un `Employee`. **C'est ici que le rôle contextuel est stocké.**

| Champ | Type | Description |
|---|---|---|
| project_id | UUID | FK → Project |
| employee_id | UUID | FK → Employee |
| role | enum | `chef` \| `monteur` |

> Un même `Employee` peut être `chef` sur le Projet A et `monteur` sur le Projet B. Le rôle n'est pas global — il ne vit pas sur `User`.

---

### WorkEntry (Pointage journalier)
Saisie de travail d'un employé pour un jour donné sur un chantier.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| project_id | UUID | FK → Project |
| employee_id | UUID | FK → Employee |
| date | date | |
| work_type_id | UUID | FK → WorkType |
| km | Decimal? | Kilomètres parcourus |
| lunch | boolean | Repas midi ? |
| dinner | boolean | Repas soir ? |

Relations : `time_ranges[]` (1-N TimeRange)

---

### TimeRange
Plage horaire dans un pointage. Plusieurs plages possibles par jour et par employé.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| work_entry_id | UUID | FK → WorkEntry |
| start_time | time | |
| end_time | time | |
| gap_acknowledged | boolean | L'utilisateur a confirmé le trou horaire |

> La détection de trou horaire est calculée en backend sur les plages d'une même journée pour un employé donné.

---

### WorkType (Type de travail)
Catalogue configurable par l'Owner. Exemples : Montage, Fachzeit, Fahrzeit.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| code | string | Identifiant court |
| label_fr | string | Libellé français |
| label_de | string | Libellé allemand |
| is_billable | boolean | Facturable au client ? |
| requires_km | boolean | Nécessite la saisie de km ? |
| color | string | Couleur d'affichage (hex) |

---

### HardwareCategory & HardwareItem (Quincaillerie)
Catalogue d'articles configurable par l'Owner.

**HardwareCategory**

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| name | string | Ex : "Charnières", "Glissières" |

**HardwareItem**

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| category_id | UUID | FK → HardwareCategory |
| ref | string | Référence article |
| name | string | |
| unit | string | Ex : "pièce", "mètre", "boîte" |
| unit_price | Decimal | Prix catalogue |

---

### ProjectHardware
Articles de quincaillerie utilisés sur un chantier. Le prix unitaire est snapshoté à la saisie.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| project_id | UUID | FK → Project |
| hardware_item_id | UUID | FK → HardwareItem |
| quantity | integer | |
| multiplier | integer | Multiplicateur ×N (meubles répétés) |
| unit_price_snapshot | Decimal | Prix au moment de la saisie |

---

### Hotel
Informations hôtel pour un chantier. Souvent saisie tard, à la clôture.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| project_id | UUID | FK → Project (1-1) |
| name | string | |
| address | string? | |
| price_per_night | Decimal | |
| nights | integer | |

---

### Expense (Dépense)
Dépense avec justificatif photo.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| project_id | UUID | FK → Project |
| type | string | Ex : "Parking", "Restaurant", "Hôtel" |
| amount | Decimal | |
| file_url | string? | URL vers le justificatif stocké |
| date | date | |

---

### Incident (Aléa)
Temps perdu à cause d'un autre corps de métier. Génère un document de notification.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| project_id | UUID | FK → Project |
| description | text | |
| duration_lost | integer | Durée en minutes |
| date | date | |
| photo_urls | string[] | URLs vers les photos |

---

### InvoiceDossier
Dossier de facturation généré à la clôture d'un chantier.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| project_id | UUID | FK → Project (1-1) |
| organization_id | UUID | FK → Organization |
| status | enum | `draft` \| `validated` |
| created_at | timestamp | |
| validated_at | timestamp? | |

Relations : `documents[]` (1-N InvoiceDocument)

---

### InvoiceDocument
Document individuel dans un dossier : Regie-Rapport, Verbrauchsmaterial, facture client, facture intérim.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| dossier_id | UUID | FK → InvoiceDossier |
| type | enum | `regie` \| `verbrauch` \| `client` \| `interim` |
| year | integer | Année de facturation |
| pdf_url | string? | URL du PDF généré |

Relations : `lines[]` (1-N InvoiceLine)

---

### InvoiceLine
**Snapshot immuable.** Ligne de facturation générée à partir des données terrain et des tarifs au moment de la clôture.

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| document_id | UUID | FK → InvoiceDocument |
| description | string | Libellé de la ligne |
| quantity | Decimal | |
| unit_price | Decimal | **Prix au moment de la génération — jamais recalculé** |
| total | Decimal | quantity × unit_price |
| vat_rate | Decimal? | Taux TVA appliqué (question ouverte — voir Q2) |
| currency | string | `CHF` par défaut |
| sort_order | integer | Ordre d'affichage |

> Ces lignes ne sont jamais recalculées depuis les tarifs courants. Voir [docs/00-mvp-decisions.md](./00-mvp-decisions.md) — Décision 2.

---

### Vehicle & VehicleRate
Type de véhicule configurable, avec prix au km par client.

**Vehicle**

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization |
| type | string | Ex : "T1", "T2", "T3" |
| label | string | |
| description | string? | |

**VehicleRate**

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| vehicle_id | UUID | FK → Vehicle |
| client_id | UUID | FK → Client |
| price_per_km | Decimal | |

---

### CompanyInfo
Informations de l'entreprise pour les documents officiels (adresse, signature, coordonnées bancaires).

| Champ | Type | Description |
|---|---|---|
| id | UUID | |
| organization_id | UUID | FK → Organization (1-1) |
| name | string | |
| address | text | |
| phone | string? | |
| email | string? | |
| vat_number | string? | Numéro TVA |
| bank_details | JSON? | IBAN, BIC |
| signature_url | string? | Image de signature |
| logo_url | string? | Logo |

---

## Diagramme des relations principales

```
Organization
├── User (org_id)
│   └── [lié à] Employee
├── Employee (org_id)
├── Client (org_id)
│   ├── Contact[]
│   ├── ClientTariff (1-1)
│   └── [véhicules autorisés] → Vehicle[]
├── Project (org_id, client_id)
│   ├── ProjectMember[] → Employee (role: chef|monteur)
│   ├── WorkEntry[] → Employee, WorkType
│   │   └── TimeRange[]
│   ├── ProjectHardware[] → HardwareItem
│   ├── Hotel? (1-1)
│   ├── Expense[]
│   ├── Incident[]
│   └── InvoiceDossier? (1-1)
│       └── InvoiceDocument[]
│           └── InvoiceLine[] ← snapshot immuable
├── WorkType (org_id)
├── HardwareCategory (org_id)
│   └── HardwareItem[] (org_id)
├── Vehicle (org_id)
│   └── VehicleRate[] → Client
└── CompanyInfo (org_id, 1-1)
```

---

## Implications multi-tenant — règles de développement

### Règle 1 — Tout filtrer par organization_id
Chaque requête touchant une entité métier doit inclure un filtre `WHERE organization_id = :orgId`. Sans ce filtre, les données d'un tenant peuvent être exposées à un autre.

### Règle 2 — Jamais de cross-tenant
Aucune jointure ne doit traverser les données de deux tenants distincts. Les routes NestJS injectent l'`organization_id` depuis le token JWT, jamais depuis le payload client.

### Règle 3 — Catalogues par tenant
`WorkType`, `HardwareItem`, `HardwareCategory`, `Vehicle`, `CompanyInfo` sont des données propres à chaque `Organization`. Il n'y a pas de catalogue global partagé.

### Règle 4 — Isolation applicative en MVP
En MVP, l'isolation est garantie au niveau applicatif (filtre dans chaque query). PostgreSQL Row Level Security (RLS) peut être ajouté en option future sans changement de modèle.

---

*Greder Montagen — mini ERP Suisse · Modèle de domaine · 7 juin 2026*
