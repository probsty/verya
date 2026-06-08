# ADR 0006 — Docker Compose for Local Development

## Status

Accepted

## Context

L'environnement de développement doit être reproductible et démarrable en une seule commande. Le projet nécessite PostgreSQL, un stockage compatible S3 pour les fichiers (photos de chantier, justificatifs, PDFs), et potentiellement d'autres services tiers simulés localement.

Les développeurs ne doivent pas avoir à installer PostgreSQL localement ni à configurer un stockage cloud pour travailler.

## Decision

L'environnement de développement local est géré par **Docker Compose** dans le dossier `infra/`.

### Services initiaux

```yaml
# Ordre de priorité pour la Phase 1
services:
  postgres:    # PostgreSQL — obligatoire dès Phase 1
  minio:       # S3-compatible local — obligatoire avant Phase 4 (upload fichiers)
  pgadmin:     # Interface PostgreSQL — optionnel, non bloquant
```

`docker compose up` doit être la seule commande nécessaire pour démarrer l'environnement de développement.

### Secrets et variables d'environnement

- Les variables d'environnement sensibles (mots de passe DB, clés MinIO, secrets JWT) sont définies dans un fichier `.env` à la racine du projet
- `.env` est dans `.gitignore` — jamais commité
- `.env.example` est versionné dans git avec des valeurs fictives documentant les variables requises
- Aucun secret n'est hardcodé dans le code ou dans `docker-compose.yml`

### Dockerisation des applications

La dockerisation de `apps/api` et `apps/web` est prévue mais n'est pas prioritaire pour la Phase 1. Elle sera mise en place lors de la préparation au déploiement (Phase 5 ou post-MVP).

### Hébergement production

Les contraintes de souveraineté des données (nLPD) imposent un hébergement en Suisse ou dans l'UE. Options identifiées : Exoscale (CH), Infomaniak (CH). Décision finale lors de la préparation au déploiement.

## Alternatives considered

**PostgreSQL installé localement** — plus léger, pas de Docker nécessaire. Écarté car non reproductible entre machines et versions, et ne couvre pas le stockage S3 local.

**Supabase local** — fournit PostgreSQL + storage + auth en local. Écarté car ajoute des services non utilisés (auth Supabase, Realtime) et crée une dépendance à un écosystème spécifique.

**Docker Desktop seul (sans Compose)** — nécessite des commandes manuelles pour chaque service. Écarté au profit de Compose pour la reproductibilité.

## Consequences

- **Positif :** environnement de développement reproductible en une commande.
- **Positif :** MinIO local émule exactement le comportement S3 de production sans coût.
- **Positif :** `.env.example` documente les variables requises et facilite l'onboarding.
- **Négatif :** Docker Desktop requis sur la machine de développement.
- **Vigilance :** ne jamais commiter `.env` — vérifier que `.gitignore` est correctement configuré avant le premier commit contenant des credentials.
- **Vigilance :** les variables d'environnement de production ne doivent jamais transiter par le repo git — utiliser les secrets du CI/CD ou un gestionnaire de secrets dédié.
- **Vigilance :** pgAdmin est optionnel — son absence ne doit pas bloquer le démarrage du projet.

---

*Veyra — Greder Montagen · ADR 0006 · Juin 2026*
