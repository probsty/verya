# ADR 0002 — Angular Frontend

## Status

Accepted

## Context

Le frontend doit couvrir deux contextes d'usage très différents : la saisie terrain sur smartphone par les chefs monteurs (mobile-first, saisie rapide, conditions difficiles) et la gestion administrative sur desktop par le Business Owner (tableaux, formulaires complexes, génération de documents).

L'interface doit être bilingue FR/DE dès le départ. Un troisième langue (italien, Tessin) est envisageable à moyen terme.

Le prototype existant est en React + localStorage. La réécriture vise une application multi-utilisateur robuste avec un cadre structurant.

## Decision

Le frontend est développé avec **Angular** en utilisant **Angular CLI** pour la génération initiale et les composants.

Structure des dossiers dans `apps/web/src/app/` :

```
app/
├── core/         # Services singleton, interceptors, guards, modèles globaux
├── shared/       # Composants, pipes et directives réutilisables
├── features/     # Modules métier (clients, projects, invoicing, etc.)
│   ├── clients/
│   ├── employees/
│   ├── projects/
│   ├── time-tracking/
│   ├── invoicing/
│   └── settings/
├── layout/       # Shell de l'application (sidebar, header, wrappers de page)
└── app.routes.ts
```

Règles complémentaires :
- **Standalone components** — cohérent avec Angular 17+ ; pas de NgModules par feature sauf nécessité explicite
- **Lazy loading** — chaque feature est chargée à la demande pour préserver les performances mobile
- **Mobile-first** — les vues terrain (pointage, frais, quincaillerie) sont conçues pour smartphone en premier
- **i18n FR/DE** — structure préparée dès le départ (fichiers de traduction, `TranslateService` ou `@angular/localize`). La décision de librairie i18n sera tranchée en tenant compte de la probabilité d'ajout de l'italien (voir `docs/06-open-questions.md` — Q9)
- **Loading / Empty / Error states** — chaque liste et formulaire gère les trois états
- **URL API** — jamais hardcodée dans les composants, injectée via environment ou token
- **Design system** — pas de bibliothèque de composants génériques avant que les patterns UI soient stabilisés sur au moins deux features

## Alternatives considered

**React** — plus flexible, écosystème plus large. Écarté car le prototype existant est déjà en React et a montré ses limites sur la structuration (tout dans localStorage, pas de conventions d'architecture). Angular impose une structure qui convient mieux à une application métier complexe.

**Vue.js / Nuxt** — bonne alternative, mais Angular offre un cadre plus directif (DI, modules, conventions CLI) adapté au profil du projet.

**SvelteKit** — léger et performant. Écarté car l'écosystème enterprise et la maturité des outils Angular (DevTools, CLI, forms réactifs) sont plus adaptés à ce type d'ERP.

## Consequences

- **Positif :** conventions fortes imposées par Angular CLI réduisent les décisions d'architecture au quotidien.
- **Positif :** Reactive Forms d'Angular sont bien adaptés aux formulaires complexes de saisie terrain.
- **Positif :** lazy loading natif, tree-shaking efficace.
- **Négatif :** bundle initial plus lourd que React ou Vue si non optimisé — lazy loading obligatoire pour les features.
- **Vigilance :** surveiller les performances sur mid-range Android (Chrome DevTools throttling 4G).
- **Vigilance :** Angular CLI doit être utilisé pour générer composants, services et guards — ne pas créer ces fichiers manuellement.
- **Vigilance :** ne pas extraire de composants partagés avant qu'un pattern soit utilisé dans au moins deux features et stabilisé.

---

*Veyra — Greder Montagen · ADR 0002 · Juin 2026*
