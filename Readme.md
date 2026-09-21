# Angular

Ce cours couvre l'ensemble des notions d'Angular en construisant **Pokedev**, un pokédex de développeurs.

Chaque dev a un numéro, un ou deux types (front-end, back-end, DevOps, data, mobile, sécurité), six statistiques, des langages et parfois une évolution (Stagiairon → Juniorax → Seniorgon). L'utilisateur parcourt le pokédex, consulte les fiches, compose une équipe de six devs et crée ses propres devs.

Il n'y a rien à télécharger pour commencer : le projet Pokedev est créé puis complété pas à pas, phase après phase.

Le cours se termine par **Time To**, un projet à réaliser en autonomie : une application qui trouve le meilleur créneau de la semaine pour une activité d'extérieur.

---

## Sommaire

| Étape | Fichier | Contenu |
|---|---|---|
| 01 | [Introduction aux frameworks](01-introduction-frameworks.md) | Bibliothèque et framework, principes et historique d'Angular, comparaison avec React, Vue et Svelte |
| 02 | [TypeScript](02-typescript.md) | Types, interfaces, unions, narrowing, génériques, classes, type guards |
| 03 | [Les outils et la création du projet](03-environnement-tooling.md) | Outils du poste, CLI Angular, outils d'aide, création et découverte du projet |
| 04 | [Conception](04-conception.md) | Découpage en composants, entrées et sorties, user stories et critères d'acceptation en Gherkin |
| 05 | [Composants et signaux](05-composants-signaux.md) | Templates, liaisons, control flow, `signal`, `computed`, `input()`, `output()`, projection de contenu |
| 06 | [Services, injection et HTTP](06-services-http.md) | `@Service()`, `inject()`, `InjectionToken`, `httpResource`, validation, intercepteur |
| 07 | [Directives et pipes](07-directives-pipes.md) | Directive d'attribut, liaisons d'hôte, `hostDirectives`, pipes personnalisés et intégrés, locale |
| 08 | [Routing](08-routing.md) | Routes, chargement différé, paramètres, gardes, resolver, `linkedSignal`, `@defer` |
| 09 | [État, persistance et architecture](09-etat-persistance-architecture.md) | État partagé, `effect`, `localStorage`, architecture en couches, fonctions pures |
| 10 | [Formulaires](10-formulaires.md) | Signal Forms, validation, validation croisée, debounce, `viewChild`, garde de sortie |
| 11 | [Tests](11-tests.md) | Vitest, tests unitaires, tests de services, de requêtes HTTP et de composants, couverture |
| 12 | [Sécurité, build et déploiement](12-securite-build-deploiement.md) | XSS, CSP, build de production, CI/CD, GitHub Pages, lecture de code historique |
| 13 | [Projet Time To](13-projet-time-to.md) | Cahier des charges du projet réalisé en autonomie |

| Ressource | Contenu |
|---|---|
| [pokedev-corriges.zip](pokedev-corriges.zip) | Les corrigés : le projet Pokedev tel qu'il doit être à la fin de chaque phase (`fin-phase-03` à `fin-phase-12` ; la phase 04 ne produit pas de code). À n'ouvrir qu'en cas de blocage, ou pour reprendre le cours à une étape précise. Dans un dossier : `npm install`, puis `npx ng serve`. |
| [time-to-corrige.zip](time-to-corrige.zip) | Une réalisation possible du projet Time To. |

---

## Versions

Angular 22.1, Angular CLI 22.1, TypeScript 6.0 (installé par la CLI), Vitest 4, Node.js 24 LTS.

Angular publie une version majeure environ tous les six mois. La commande `npm view @angular/cli version` affiche la version courante.

---

## Prérequis

- **Node.js 22.22.3 ou plus, ou 24.15 ou plus.** La CLI Angular 22 refuse les versions antérieures.
- Git et un compte GitHub.
- VS Code avec l'extension *Angular Language Service*.
- Un navigateur Chromium ou Firefox avec l'extension *Angular DevTools*.

---

[Commencer : 01 · Introduction aux frameworks →](01-introduction-frameworks.md)
