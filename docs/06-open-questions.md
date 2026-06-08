# Questions ouvertes — mini ERP Suisse (Greder Montagen)

**Produit :** ERP Montage / Chantiers  
**Date :** 7 juin 2026

---

## Décisions déjà prises

Les questions suivantes ont été résolues lors de la phase de documentation. Voir [docs/00-mvp-decisions.md](./00-mvp-decisions.md) pour le détail complet.

| Question | Décision |
|---|---|
| Mono-entreprise ou multi-tenant ? | Multi-tenant avec `Organization` dès le départ |
| Tarifs figés ou snapshot par facture ? | Snapshot à la génération — indépendant des tarifs courants |
| Workflow de validation owner ? | Owner valide toujours la facture finale |
| Le monteur a-t-il besoin d'un compte ? | Pas dans le MVP — Employee sans User |
| Mode hors-ligne dans le MVP ? | Non — connexion internet obligatoire |

---

## Questions restantes

Ces questions n'ont pas encore de réponse validée. Elles sont classées par phase de blocage.

---

### Q1 — Intégration Bexio

**Bloquant pour :** Post-MVP  
**Question :** L'intégration Bexio est-elle un export ponctuel (PDF/CSV téléchargeable) ou une synchronisation API bidirectionnelle ?

- **Export ponctuel :** l'Owner exporte un fichier depuis l'app, l'importe dans Bexio manuellement — simple, pas de dépendance technique forte
- **Sync API :** l'app pousse automatiquement les factures validées dans Bexio via son API — nécessite gestion des credentials par tenant, job planifié, gestion des erreurs

**Impact sur l'architecture :** La sync API ajoute une complexité significative (credentials par Organization, retry logic, statut de synchronisation sur InvoiceDocument).

---

### Q2 — TVA suisse

**Bloquant pour :** Phase 4 (moteur de facturation, `InvoiceLine.vat_rate`)  
**Question :** Greder Montagen est-elle assujettie à la TVA ?

Si oui :
- Quel taux s'applique ? (8,1% standard · 2,6% réduit · 3,8% spécial hôtellerie)
- La TVA est-elle affichée par ligne ou uniquement en pied de facture ?
- Certaines prestations sont-elles exonérées (ex : prestations de service vers l'étranger) ?

**Impact :** Le champ `InvoiceLine.vat_rate` est prévu dans le modèle. Le taux et les règles d'application doivent être confirmés avant de coder le moteur de calcul.

---

### Q3 — Arrondi suisse à 0,05 CHF

**Bloquant pour :** Phase 4 (moteur de facturation)  
**Question :** Les totaux de facture doivent-ils être arrondis à 0,05 CHF ?

En Suisse, les paiements en espèces sont arrondis à 0,05 CHF (pièce la plus petite en circulation). Cette pratique est courante sur les factures B2B.

**Options :**
- Arrondi à 0,05 CHF sur le total facture (standard CH)
- Affichage à 2 décimales (centimes), sans arrondi spécifique

**Impact :** Règle d'arrondi à implémenter dans le moteur de calcul et à afficher explicitement sur les documents. Voir aussi `docs/05-technical-risks.md` — R3.

---

### Q4 — Politique d'archivage légal

**Bloquant pour :** Infrastructure, Phase 5 ou post-MVP  
**Question :** Quelle est la politique de conservation des dossiers de facturation ?

La loi suisse (CO art. 958f) exige la conservation des pièces comptables pendant **10 ans**.

Points à préciser :
- Les dossiers validés peuvent-ils être supprimés par l'Owner, ou uniquement archivés ?
- Un export d'archive (ZIP, CSV) est-il nécessaire pour l'audit fiscal ?
- L'app doit-elle gérer une corbeille avec délai de rétention avant suppression définitive ?

---

### Q5 — Signature électronique sur le Regie-Rapport

**Bloquant pour :** Phase 4 (génération documents)  
**Question :** Le Regie-Rapport nécessite-t-il une signature électronique ?

Options :
- **Signature manuscrite scannée** stockée comme image dans `CompanyInfo.signature_url` — simple, déjà prévu dans le modèle
- **Signature électronique qualifiée** (QES selon eIDAS / ZertES) — prestataire suisse requis (SwissSign, etc.)
- **Pas de signature électronique** — impression + signature manuscrite physique du client

**Impact :** La signature qualifiée implique l'intégration d'un prestataire tiers et change l'architecture de génération des documents.

---

### Q6 — Seeding du premier tenant en production

**Bloquant pour :** Phase 2 (Prisma setup)  
**Question :** Comment créer Greder Montagen et le premier compte Owner en production ?

Options :
- Script de seeding manuel exécuté une fois (`prisma db seed` avec variables d'environnement)
- Endpoint d'initialisation sécurisé (route protégée par un secret d'admin, désactivable)
- Interface d'onboarding — hors MVP (voir Décision 1)

**Impact :** À décider avant la Phase 2 pour inclure la logique dans le script de seeding.

---

### Q7 — Devise

**Bloquant pour :** Phase 4 (moteur de facturation)  
**Question :** Les factures sont-elles uniquement en CHF, ou une facturation en EUR est-elle envisagée (clients frontaliers, France, Allemagne) ?

**Impact :** Si multi-devises, `InvoiceLine.currency` doit être utilisé activement, et les totaux de l'`InvoiceDocument` doivent gérer des devises distinctes ou une conversion.

---

### Q8 — Compte utilisateur Monteur (post-MVP)

**Bloquant pour :** Post-MVP — mais oriente les priorités  
**Question :** Les monteurs auront-ils besoin d'un compte utilisateur, et à quel horizon ?

Le modèle de données prévoit déjà cette évolution : `Employee` existe sans `User` dans le MVP, la liaison sera possible plus tard sans migration de schéma. La question est de savoir si ce besoin est à 6 mois ou 2 ans.

---

### Q9 — Troisième langue (IT — Tessin)

**Bloquant pour :** Phase 1 (choix bibliothèque i18n Angular)  
**Question :** Le délai pour l'ajout de l'italien est-il à court terme (< 6 mois) ou long terme ?

**Impact :** Si court terme, privilégier une bibliothèque i18n Angular facilitant l'ajout de locales (`ngx-translate`). Si long terme, ce choix est moins critique.

---

### Q10 — Format exact du Regie-Rapport

**Bloquant pour :** Phase 4 (génération PDF)  
**Question :** Le gabarit du Regie-Rapport est-il disponible sous forme de fichier existant (PDF, Word, InDesign) ?

**Impact :** Reproduire fidèlement un document existant requiert une approche HTML → PDF (Puppeteer). Concevoir une mise en page libre est faisable avec PDFKit (plus léger). La disponibilité du gabarit détermine le choix de la bibliothèque de génération PDF. Voir `docs/05-technical-risks.md` — R6.

---

*Greder Montagen — mini ERP Suisse · Questions ouvertes · 7 juin 2026*
