# Angular

Ce cours couvre l'ensemble des notions d'Angular en construisant **Pokedev**, un pokédex de développeurs.

Chaque dev a un numéro, un ou deux types (front-end, back-end, DevOps, data, mobile, sécurité), six statistiques, des langages et parfois une évolution (Stagiairon → Juniorax → Seniorgon). L'utilisateur parcourt le pokédex, consulte les fiches, compose une équipe de six devs et crée ses propres devs.

Il n'y a rien à télécharger pour commencer : le projet Pokedev est créé puis complété pas à pas, phase après phase.

---

## Ce que comprend le cours

- **Pokedev, le projet fil rouge.** Les chapitres 01 à 12, dans ce dépôt : chaque notion est présentée, puis mise en œuvre sur le projet.
- **La boîte à outils Angular 22.** Douze fiches, une par notion (composant, routeur, signaux, services, HTTP, tests…), et un mémo de génération de code. Chaque fiche résume l'essentiel et renvoie au chapitre du cours qui détaille la notion : à garder ouverte pendant tout le cours. Dépôt : [github.com/DiginamicAcademy/Angular22-toolbox](https://github.com/DiginamicAcademy/Angular22-toolbox)
- **Le projet de démo pas à pas.** Une application de listes de tâches, construite de zéro en douze étapes face à un petit serveur Java fourni. Chaque dossier contient le projet complet à la fin d'une étape, avec l'explication de ce qui a changé : les notions du cours sur un second exemple, plus simple. Il emploie parfois d'autres outils que Pokedev, comme `[(ngModel)]` ou `HttpClient` : des variantes à savoir lire. Dépôt : [github.com/DiginamicFormation/angular22-zero](https://github.com/DiginamicFormation/angular22-zero)
- **Time To, le projet d'entraînement, facultatif.** Une application complète à réaliser seul à partir d'un cahier des charges, pour s'exercer en autonomie après le chapitre 12. Il n'est pas noté. Chapitre : [13 · Projet Time To](13-projet-time-to.md)
- **Le TP noté.** La caisse enregistreuse d'un food truck : le serveur Java est fourni, vous réalisez l'application Angular. Le sujet est dans le fichier `ENONCE.md`. Dépôt : [github.com/DiginamicAcademy/TP-Angular22](https://github.com/DiginamicAcademy/TP-Angular22)

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
| 13 | [Projet Time To](13-projet-time-to.md) | Facultatif. Projet d'entraînement en autonomie : cahier des charges, méthode et jalons |

| Ressource | Contenu |
|---|---|
| [pokedev-corriges.zip](pokedev-corriges.zip) | Les corrigés : le projet Pokedev tel qu'il doit être à la fin de chaque phase (`fin-phase-03` à `fin-phase-12` ; la phase 04 ne produit pas de code). À n'ouvrir qu'en cas de blocage, ou pour reprendre le cours à une étape précise. Dans un dossier : `npm install`, puis `npx ng serve`. |
| [time-to-corrige.zip](time-to-corrige.zip) | Une réalisation possible du projet Time To, à comparer avec la vôtre après avoir essayé. |

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
- Pour le projet de démo et le TP noté, dont le serveur est en Java : un JDK 21 ou plus récent. Maven n'est pas nécessaire : chaque serveur le télécharge au premier lancement.

---

[Commencer : 01 · Introduction aux frameworks →](01-introduction-frameworks.md)
