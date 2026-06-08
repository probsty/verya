# Modèle de permissions — mini ERP Suisse (Greder Montagen)

**Produit :** ERP Montage / Chantiers  
**Date :** 7 juin 2026

---

## Rôles MVP

### Business Owner
- Rôle global à l'`Organization`
- Stocké sur `User.role = 'owner'`
- Accès complet à toutes les données et actions
- Seul rôle pouvant valider les factures

### Chef monteur
- Rôle **contextuel par projet**
- Stocké dans `ProjectMember.role = 'chef'`
- Un `User` peut être chef monteur sur certains projets et absent d'autres
- À la connexion, un Chef monteur voit uniquement les projets où `ProjectMember.role = 'chef'` pour son `employee_id`
- Ne valide pas les factures

### Monteur
- **Hors MVP** — pas de compte utilisateur
- Existe comme entité `Employee` assignée à des projets (`ProjectMember.role = 'monteur'`)
- N'accède pas à l'application dans le MVP

---

## Principe clé : rôle contextuel

Le rôle Chef monteur n'est pas un champ sur `User`. C'est une propriété de la relation `ProjectMember`.

```
User.role = 'owner'            → Owner global : accès total
User.role = 'member'           → Utilisateur standard : accès selon ses ProjectMember
ProjectMember.role = 'chef'    → Chef monteur sur CE projet
ProjectMember.role = 'monteur' → Monteur sur CE projet (sans accès app en MVP)
```

Un même utilisateur peut avoir le rôle `chef` sur 3 projets et être assigné `monteur` sur 2 autres (sans accès à ces derniers dans le MVP).

---

## Matrice des permissions

| Écran / Action | Business Owner | Chef monteur |
|---|---|---|
| **Connexion** | ✅ | ✅ |
| **Tableau de bord** | ✅ Tous les chantiers | ✅ Ses chantiers uniquement |
| **Clients — liste** | ✅ | ❌ |
| **Clients — créer / modifier** | ✅ | ❌ |
| **Équipe — liste** | ✅ | ❌ |
| **Équipe — créer / modifier** | ✅ | ❌ |
| **Chantiers — liste** | ✅ Tous | ✅ Ses chantiers |
| **Chantiers — créer** | ✅ | ❌ |
| **Chantiers — modifier (équipe, dates)** | ✅ | ❌ |
| **Pointage — saisir** | ✅ | ✅ (ses chantiers) |
| **Frais (km, repas) — saisir** | ✅ | ✅ (ses chantiers) |
| **Quincaillerie — saisir** | ✅ | ✅ (ses chantiers) |
| **Hôtel — saisir** | ✅ | ✅ (ses chantiers) |
| **Dépenses — saisir** | ✅ | ✅ (ses chantiers) |
| **Aléas — saisir** | ✅ | ✅ (ses chantiers) |
| **Clôture chantier** | ✅ | ✅ (ses chantiers) |
| **Factures — liste** | ✅ | ❌ |
| **Factures — modifier lignes (draft)** | ✅ | ❌ |
| **Factures — valider** | ✅ | ❌ |
| **Factures — exporter PDF** | ✅ | ❌ |
| **Administration (catalogues)** | ✅ | ❌ |
| **Profil / Réglages personnels** | ✅ | ✅ |

---

## Workflow de validation des factures

```
Chef monteur
  └── Clôture le chantier
          ↓
Système
  ├── Verrouille le projet (status: closed)
  └── Génère InvoiceDossier (status: draft)
          ↓
Business Owner
  ├── Consulte le dossier de facturation
  ├── Corrige les lignes si nécessaire (modifiable en draft)
  └── Valide → status: validated (immuable)
```

La clôture d'un chantier est irréversible. L'`InvoiceDossier` en `draft` reste modifiable par l'Owner jusqu'à validation.

---

## Règles d'autorisation NestJS (description fonctionnelle)

> Ces règles décrivent le comportement attendu. Le code correspondant sera implémenté en Phase 5.

**Guard 1 — Token valide + appartenance Organisation**  
Toute requête authentifiée doit présenter un JWT valide dont le claim `organization_id` correspond à l'organisation de la ressource demandée. L'`organization_id` est toujours lu depuis le token, jamais depuis le payload client.

**Guard 2 — Owner**  
Vérifie `User.role === 'owner'` pour les routes réservées à l'Owner (Clients, Équipe, Administration, Factures, création de chantier).

**Guard 3 — Chef monteur sur projet spécifique**  
Vérifie l'existence d'une entrée `ProjectMember WHERE project_id = :id AND employee_id = :employeeId AND role = 'chef'` pour les actions terrain sur un projet donné.

**Guard 4 — Owner OU Chef monteur**  
Pour les actions communes (ex : lecture d'un projet), vérifie Owner OU ProjectMember chef. L'Owner n'a pas besoin d'être `ProjectMember` pour accéder à un projet.

---

*Greder Montagen — mini ERP Suisse · Modèle de permissions · 7 juin 2026*
