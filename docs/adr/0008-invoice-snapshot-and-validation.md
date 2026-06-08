# ADR 0008 — Invoice Snapshot and Validation Workflow

## Status

Accepted

## Context

Les factures de Greder Montagen sont des documents officiels soumis à la loi suisse sur la comptabilité (CO art. 958f — conservation 10 ans). Les tarifs clients peuvent évoluer d'un chantier à l'autre. Si une facture recalcule ses montants depuis les tarifs courants au moment de la consultation, une modification tarifaire ultérieure changerait rétroactivement les montants affichés sur une facture déjà envoyée — scénario inacceptable légalement et commercialement.

Par ailleurs, le Business Owner doit pouvoir contrôler et corriger les données terrain avant d'envoyer une facture au client. Le Chef monteur ne doit pas avoir le pouvoir de valider une facture finale.

## Decision

### Principe : la facture est un snapshot immuable

À la clôture d'un chantier, le système génère un `InvoiceDossier` en copiant les données terrain et les tarifs du moment dans des `InvoiceLine`. Ces lignes ne sont jamais recalculées à partir des référentiels tarifaires courants.

```
InvoiceLine.unit_price  = copie du ClientTariff.hourly_rate au moment de la génération
InvoiceLine.quantity    = heures/km/nuitées calculées depuis les données terrain
InvoiceLine.total       = quantity × unit_price (calculé à la génération, stocké)
```

**Aucun JOIN entre `InvoiceLine` et `ClientTariff` dans les requêtes de lecture de facture.**

### Workflow de validation

```
Chef monteur
  └── Clôture le chantier (Project.status : active → closed)
          ↓
Système (backend)
  ├── Verrouille le projet (irréversible)
  └── Génère InvoiceDossier (status: draft)
      └── Calcule et persiste les InvoiceLine (snapshot)
          ↓
Business Owner
  ├── Consulte le dossier de facturation
  ├── Peut modifier les lignes (modifiable en draft uniquement)
  │     ⚠ La modification est manuelle — les ClientTariff courants ne sont jamais réappliqués automatiquement
  └── Valide → InvoiceDossier.status : validated (immuable)
```

### États de l'InvoiceDossier

| Status | Description | Modifiable |
|---|---|---|
| `draft` | Généré à la clôture, en attente de validation Owner | Oui (Owner uniquement) |
| `validated` | Validé par l'Owner | Non — immuable |

### Données snapshotées dans chaque InvoiceLine

- Description de la ligne
- Quantité
- Prix unitaire **au moment de la génération**
- Total (quantité × prix unitaire)
- Taux de TVA appliqué (à confirmer — voir `docs/06-open-questions.md` — Q2)
- Devise (`CHF` par défaut)
- Ordre d'affichage

### Auditabilité

- `InvoiceDossier.created_at` — horodatage de génération
- `InvoiceDossier.validated_at` — horodatage de validation
- Les `InvoiceLine` ne doivent jamais être supprimées après validation
- La politique d'archivage long terme (10 ans, CO art. 958f) est à définir (voir `docs/06-open-questions.md` — Q4)

## Alternatives considered

**Recalcul dynamique depuis les tarifs courants** — affichage toujours à jour des montants. Écarté car une modification des `ClientTariff` changerait rétroactivement les montants affichés sur des factures déjà envoyées — inacceptable légalement et commercialement.

**Versioning des tarifs (ClientTariffHistory)** — conserver l'historique des changements de tarifs et recalculer avec le tarif applicable à la date du chantier. Plus complexe à implémenter, potentiellement plus flexible. Écarté pour le MVP au profit du snapshot direct, plus simple et tout aussi correct.

**Validation par le Chef monteur** — permettrait plus d'autonomie au Chef. Écarté car le Business Owner doit contrôler les données financières avant envoi au client. La clôture (acte terrain) et la validation (acte financier) sont deux responsabilités distinctes.

## Consequences

- **Positif :** intégrité des données financières garantie — une facture validée ne change jamais.
- **Positif :** conformité avec les obligations comptables suisses (CO art. 958f).
- **Positif :** l'Owner peut corriger des erreurs de saisie terrain avant validation, sans recalcul automatique intempestif.
- **Négatif :** les `InvoiceLine` sont des données dénormalisées — duplication intentionnelle des tarifs au moment de la génération.
- **Vigilance :** risque identifié dans `docs/05-technical-risks.md` — R2. Tests unitaires du moteur de calcul obligatoires : vérifier que les lignes générées sont indépendantes des tarifs courants après modification des `ClientTariff`.
- **Vigilance :** les calculs de facturation (heures × tarif, km, repas, hôtel, quincaillerie, intérim) se font **côté backend**. Aucun calcul de montant dans les controllers Angular.
- **Vigilance :** la stratégie monétaire (Decimal, arrondi CHF 0,05) doit être implémentée avant le moteur de facturation (Phase 4).

---

*Veyra — Greder Montagen · ADR 0008 · Juin 2026*
