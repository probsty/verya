# Risques techniques — mini ERP Suisse (Greder Montagen)

**Produit :** ERP Montage / Chantiers  
**Date :** 7 juin 2026

---

## Risques élevés 🔴

### R1 — Fuite de données cross-tenant

**Contexte :** L'isolation multi-tenant est appliquée au niveau applicatif (filtre `WHERE organization_id = ?` dans chaque query NestJS). Un oubli de ce filtre expose les données d'un tenant aux requêtes d'un autre.

**Impact :** Fuite de données confidentielles (clients, tarifs, factures) entre entreprises. Violation nLPD.

**Mitigation :**
- Convention de code systématique : toute query Prisma sur une entité métier inclut `organizationId`
- Helper NestJS injectant automatiquement l'`organizationId` depuis le contexte de requête (token JWT)
- Tests d'intégration spécifiques : vérifier qu'un tenant A ne peut jamais lire les données d'un tenant B
- Option future : PostgreSQL Row Level Security comme filet de sécurité supplémentaire (sans changement de modèle)

---

### R2 — Intégrité des snapshots de facturation

**Contexte :** Les `InvoiceLine` stockent les prix au moment de la génération. Si une query recalcule les totaux depuis les `ClientTariff` courants, les vieilles factures affichent de mauvais montants.

**Impact :** Erreurs de facturation, litiges clients, perte de confiance.

**Mitigation :**
- `InvoiceLine.unit_price` est écrit une seule fois à la génération, jamais mis à jour
- Aucun `JOIN` entre `InvoiceLine` et `ClientTariff` dans les requêtes de lecture de facture
- Tests unitaires du moteur de calcul vérifiant que les lignes générées sont indépendantes des tarifs courants
- L'Owner peut modifier les lignes en `draft`, mais les `ClientTariff` courants ne sont jamais réappliqués automatiquement

---

### R3 — Précision monétaire (float vs Decimal)

**Contexte :** Les calculs financiers (heures × tarif, sommes de lignes) peuvent accumuler des erreurs d'arrondi si réalisés avec des types `float` ou `double` JavaScript.

**Impact :** Totaux erronés sur les factures. Différences de centimes s'accumulant sur plusieurs lignes.

**Mitigation :**
- Type `Decimal` dans Prisma → `NUMERIC` dans PostgreSQL pour tous les champs montants
- Bibliothèque `decimal.js` ou `big.js` pour les calculs côté Node.js
- Aucun `number` JavaScript natif pour les calculs financiers
- La règle d'arrondi suisse (0,05 CHF) sera appliquée sur le total facture une fois confirmée (voir `docs/06-open-questions.md` — Q3)

---

### R4 — Sécurité & conformité nLPD

**Contexte :** L'application gère des données personnelles de clients suisses (contacts, adresses, tarifs). La nLPD (nouvelle Loi sur la Protection des Données) impose des obligations de protection et de localisation.

**Impact :** Violation légale, sanctions, perte de confiance client.

**Mitigation :**
- Mots de passe hachés avec bcrypt (coût ≥ 12)
- HTTPS obligatoire en production
- Hébergement en Suisse ou UE (Exoscale CH, Infomaniak, ou équivalent)
- Chiffrement des données au repos (disque chiffré au niveau infra)
- Pas de logs exposant des PII (noms, e-mails, montants)
- Politique de conservation des données à définir (voir `docs/06-open-questions.md` — Q4)

---

## Risques moyens 🟠

### R5 — Bug d'accès RBAC contextuel

**Contexte :** Le rôle Chef monteur est contextuel par projet (stocké dans `ProjectMember`). Une vérification insuffisante permettrait à un Chef monteur d'accéder aux données d'un projet dont il n'est pas responsable, ou aux écrans Owner (Clients, Factures).

**Impact :** Exposition de données financières et clients à des personnes non autorisées.

**Mitigation :**
- Guards NestJS distincts : `OwnerGuard` (vérifie `User.role === 'owner'`) et `ProjectChefGuard` (vérifie `ProjectMember`)
- L'`organization_id` et l'`employee_id` sont toujours lus depuis le token JWT, jamais depuis le payload client
- Tests d'autorisation systématiques : chaque route doit avoir un test "accès interdit" pour les rôles non autorisés

---

### R6 — Génération PDF instable en Docker

**Contexte :** Puppeteer (Chrome headless) est la solution courante pour générer des PDFs HTML côté serveur, mais est notoire pour sa complexité en Docker (dépendances système, mémoire, timeouts, crashes).

**Impact :** Génération PDF échouant en production, dossiers de facturation non générables.

**Mitigation :**
- Évaluer **PDFKit** (génération programmatique, léger, sans navigateur) si les gabarits sont simples
- Si Puppeteer est nécessaire pour la fidélité HTML → PDF, l'isoler dans un container dédié séparé du backend NestJS
- Tester la génération PDF dès la Phase 4, pas en dernier
- Décision dépend du gabarit disponible (voir `docs/06-open-questions.md` — Q10)

---

### R7 — Stockage fichiers et souveraineté des données

**Contexte :** Les photos de chantier, justificatifs et tickets contiennent des informations commerciales sensibles. Leur stockage hors CH/UE violerait la nLPD.

**Impact :** Violation nLPD. Perte de données si stockage local non sauvegardé.

**Mitigation :**
- Stockage S3-compatible hébergé en Suisse ou UE : **Exoscale Object Storage** (CH), **Infomaniak**, ou **MinIO** auto-hébergé
- Pas d'AWS S3 us-east-1 ni de solutions hors CH/UE
- MinIO dans Docker Compose pour le développement local
- URLs signées avec expiration pour l'accès (pas de fichiers publics)

---

### R8 — Migrations Prisma en production

**Contexte :** Les migrations de schéma Prisma doivent être appliquées de manière contrôlée. `prisma db push` peut entraîner des pertes de données. Des migrations mal gérées cassent la production.

**Impact :** Perte de données, downtime, schéma incohérent.

**Mitigation :**
- Jamais de `prisma db push` en production — uniquement `prisma migrate deploy`
- Migrations versionnées dans git, revue avant merge
- Migrations testées sur une copie de la base avant déploiement
- Une PR = une migration au maximum

---

## Risques faibles 🔵

### R9 — Performance Angular sur mobile

**Contexte :** L'application doit fonctionner sur des smartphones de gamme moyenne utilisés sur chantier. Un bundle Angular trop lourd dégradera l'expérience terrain.

**Impact :** Adoption terrain insuffisante, saisies tardives ou abandonnées.

**Mitigation :**
- Lazy-loading de tous les modules Angular (pas de bundle monolithique)
- Test de performance sur mid-range Android (Chrome DevTools, throttling 4G)
- Éviter les dépendances lourdes inutiles dans le bundle principal

---

### R10 — Concurrence sur les données de projet

**Contexte :** L'Owner et un Chef monteur peuvent modifier le même projet simultanément, entraînant des écrasements silencieux.

**Impact :** Perte de données saisies, incohérences dans les pointages.

**Mitigation :**
- Champ `version` (integer) sur l'entité `Project` pour l'optimistic locking
- La requête de mise à jour vérifie `WHERE id = ? AND version = ?` et incrémente `version`
- En cas de conflit : retourner HTTP 409, demander à l'utilisateur de recharger

---

*Greder Montagen — mini ERP Suisse · Risques techniques · 7 juin 2026*
