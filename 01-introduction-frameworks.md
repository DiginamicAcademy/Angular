[← Sommaire](README.md)

---

# Phase 01 — Introduction aux frameworks front-end

## Objectifs

- Distinguer une bibliothèque d'un framework.
- Expliquer le problème qu'un framework front-end résout.
- Citer les principes propres à Angular.
- Situer Angular par rapport à React, Vue et Svelte.

---

## On théorise

### 1. Bibliothèque ou framework ?

- Une **bibliothèque** est un outil que *votre* code appelle quand il en a besoin. Exemple : une bibliothèque de dates.
- Un **framework** fonctionne à l'inverse : c'est *lui* qui structure l'application et appelle votre code au bon moment. On parle d'**inversion de contrôle**. Le framework impose une architecture, des conventions et un cycle de vie.

Les frameworks front-end modernes partagent trois idées :

| Idée | Signification |
|---|---|
| **Composants** | L'interface est un arbre de briques réutilisables. Chacune regroupe structure, style et comportement. |
| **Rendu déclaratif** | On décrit *ce que* l'écran doit afficher en fonction de l'état, pas *comment* le modifier. |
| **Réactivité** | Quand l'état change, le framework met à jour l'écran automatiquement. |

Une **SPA** (*Single Page Application*) est chargée une seule fois. Le navigateur change ensuite de « page » sans recharger le document : le framework réécrit le DOM.

### 2. Pourquoi un framework ?

En JavaScript pur, chaque modification de l'état doit être répercutée à la main dans le DOM. Enregistrez ce fichier sous `sans-framework.html` et ouvrez-le dans un navigateur :

```html
<!doctype html>
<html lang="fr">
  <head>
    <meta charset="utf-8" />
    <title>Sans framework</title>
  </head>
  <body>
    <p>Mon équipe : <span id="team-size">0/6</span></p>
    <button type="button" id="add">Ajouter un dev</button>

    <script>
      let teamSize = 0;
      const badge = document.querySelector('#team-size');
      document.querySelector('#add').addEventListener('click', () => {
        teamSize++;
        badge.textContent = `${teamSize}/6`; // à répéter partout où teamSize est affiché
      });
    </script>
  </body>
</html>
```

Dans une vraie application, la même donnée apparaît à plusieurs endroits : l'en-tête, la carte du dev, la page de l'équipe. Oublier une mise à jour désynchronise l'état et l'affichage.

Un framework apporte :
- la **synchronisation automatique** entre l'état et l'affichage ;
- une **structure commune**, qui facilite le travail en équipe et la reprise de code ;
- des **outils intégrés** : routing, HTTP, formulaires, tests ;
- un **écosystème** et une documentation.

Il a aussi un coût : une courbe d'apprentissage, du code supplémentaire livré au navigateur, une dépendance à un projet tiers et des montées de version régulières. Pour un site vitrine statique, un framework est souvent disproportionné.

### 3. Angular : les principes

*AngularJS* (2010, version 1.x) et *Angular* (depuis 2016) sont deux frameworks différents. Angular est une réécriture complète ; AngularJS n'est plus maintenu.

| Principe | En pratique |
|---|---|
| Framework **complet** | Routing, client HTTP, formulaires, tests, CLI et internationalisation sont maintenus par la même équipe (Google). |
| **TypeScript obligatoire** | Typage vérifié à la compilation, autocomplétion, refactorisations sûres. |
| **Composants = classes** | Une classe décorée par `@Component`, associée à un template HTML enrichi : `{{ }}`, `[prop]`, `(event)`, `@if`, `@for`. |
| **Signaux** | Une valeur réactive qui prévient automatiquement ce qui en dépend. |
| **Injection de dépendances** | Les services sont *fournis* aux composants par le framework, au lieu d'être instanciés à la main. |
| **Conventions fortes** | La CLI génère le code selon les conventions officielles : deux projets Angular se ressemblent. |
| **Cycle prévisible** | Une version majeure tous les six mois environ, un support long par version, des migrations automatisées avec `ng update`. |

**Angular 22** (juin 2026) rend stables les Signal Forms et `httpResource`. Les nouvelles applications fonctionnent **sans zone.js** (*zoneless*) : ce sont les signaux qui indiquent à Angular quoi mettre à jour.

Une application Angular complète, en un seul fichier : un composant, puis son démarrage avec `bootstrapApplication`.

```ts
import { Component, computed, signal } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';

@Component({
  selector: 'app-root',
  template: `
    <p>Cafés bus : {{ cups() }}</p>
    <p>Niveau d'énergie : {{ energy() }}</p>
    <button type="button" (click)="drink()">Boire un café</button>
    <button type="button" (click)="reset()">Remettre à zéro</button>
  `,
})
export class CoffeeCounter {
  protected readonly cups = signal(0);
  protected readonly energy = computed(() =>
    this.cups() === 0 ? 'endormi' : this.cups() < 4 ? 'efficace' : 'survolté',
  );

  protected drink(): void {
    this.cups.update((cups) => cups + 1);
  }

  protected reset(): void {
    this.cups.set(0);
  }
}

bootstrapApplication(CoffeeCounter);
```

### 4. Historique d'Angular

| Version | Date | Apports majeurs |
|---|---|---|
| AngularJS 1.x | 2010 | Premier framework de Google : liaison de données bidirectionnelle, contrôleurs, `$scope`. Plus maintenu depuis 2022. |
| 2 | sept. 2016 | Réécriture complète, sans compatibilité avec AngularJS : TypeScript, composants, injection de dépendances hiérarchique, RxJS. |
| 4 | mars 2017 | Numéro 3 sauté pour aligner le routeur. Code généré plus léger, `*ngIf` avec `else`. `HttpClient` en 4.3. |
| 5 | nov. 2017 | Optimisation de build, `HttpClient` remplace l'ancien module `Http`. |
| 6 | mai 2018 | `ng add` et `ng update`, fichier `angular.json`, services `providedIn: 'root'`, Angular Elements. |
| 7 | oct. 2018 | Questions interactives de la CLI, budgets de taille, défilement virtuel et glisser-déposer (CDK). |
| 8 | mai 2019 | Chargement différé des routes avec `import()`, livraison différenciée selon le navigateur, aperçu du moteur Ivy. |
| 9 | févr. 2020 | **Ivy** devient le moteur de compilation et de rendu par défaut : applications plus légères, meilleurs messages d'erreur. |
| 10 | juin 2020 | Option de projet strict. |
| 11 | nov. 2020 | Rechargement à chaud (HMR) dans la CLI, intégration des polices au build. |
| 12 | mai 2021 | Mode strict par défaut, opérateur `??` dans les templates, Webpack 5, passage de TSLint à ESLint. |
| 13 | nov. 2021 | Ivy seul (ancien moteur supprimé), fin du support d'Internet Explorer 11. |
| 14 | juin 2022 | Composants **standalone** (aperçu), formulaires réactifs typés, `inject()` généralisé, titres de routes. |
| 15 | nov. 2022 | Standalone stable, composition de directives (`hostDirectives`), gardes de routes fonctionnelles. |
| 16 | mai 2023 | **Signaux** (aperçu), entrées obligatoires, liaison des paramètres de route aux entrées, aperçu du build esbuild. |
| 17 | nov. 2023 | **Control flow** `@if` / `@for` / `@switch`, **`@defer`**, nouveau builder esbuild + Vite par défaut, site angular.dev. Puis `input()`, `output()`, `model()` en versions 17.x. |
| 18 | mai 2024 | Détection de changements **sans zone.js** (expérimentale), `@let` (18.1), contenu par défaut de `<ng-content>`. |
| 19 | nov. 2024 | Standalone par défaut, `linkedSignal` et `resource` (expérimentaux), hydratation incrémentale (aperçu). |
| 20 | mai 2025 | API de signaux stables (`effect`, `linkedSignal`, `toSignal`), zoneless en aperçu, nouveau guide de style (fichiers sans suffixe : `app.ts`), serveur MCP (20.2). |
| 21 | nov. 2025 | **Zoneless par défaut**, **Signal Forms** (expérimental), Angular Aria (aperçu), **Vitest** par défaut, `HttpClient` fourni par défaut, serveur MCP stable. |
| 22 | juin 2026 | Signal Forms, `resource` / `httpResource` et Angular Aria **stables**, décorateur `@Service()`. |

Depuis la version 14, l'évolution suit une même direction : moins de code d'infrastructure (plus de `NgModule`, plus de zone.js), une réactivité fine par signaux et un outillage plus rapide.

### 5. Angular, React, Vue, Svelte

Le même compteur de cafés dans chaque technologie. Chaque extrait est un composant complet.

**Angular** (`main.ts`, composant et démarrage)

```ts
import { Component, signal } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';

@Component({
  selector: 'app-root',
  template: `<button type="button" (click)="drink()">Cafés : {{ cups() }}</button>`,
})
export class Coffee {
  protected readonly cups = signal(0);

  protected drink(): void {
    this.cups.update((n) => n + 1);
  }
}

bootstrapApplication(Coffee);
```

**React** (`Coffee.tsx`)

```tsx
import { useState } from 'react';

export default function Coffee() {
  const [cups, setCups] = useState(0);
  return (
    <button type="button" onClick={() => setCups(cups + 1)}>
      Cafés : {cups}
    </button>
  );
}
```

**Vue** (`Coffee.vue`)

```vue
<script setup lang="ts">
import { ref } from 'vue';

const cups = ref(0);
</script>

<template>
  <button type="button" @click="cups++">Cafés : {{ cups }}</button>
</template>
```

**Svelte** (`Coffee.svelte`)

```svelte
<script lang="ts">
  let cups = $state(0);
</script>

<button type="button" onclick={() => cups++}>Cafés : {cups}</button>
```

Pour les essayer sans installation : https://angular.dev/playground, https://react.dev/learn (bacs à sable intégrés), https://play.vuejs.org et https://svelte.dev/playground.

| | Angular | React | Vue | Svelte |
|---|---|---|---|---|
| Nature | Framework complet | Bibliothèque d'interface | Framework progressif | Framework compilé |
| Version (septembre 2026) | 22 | 19.3 | 3.5 stable (3.6 en RC) | 5 |
| Templates | HTML + syntaxe Angular | JSX (HTML dans le JS) | HTML dans des fichiers `.vue` | HTML dans des fichiers `.svelte` |
| Réactivité | Signaux | État + re-rendu (hooks) | Refs réactives | Runes compilées |
| Routing, HTTP, formulaires | Inclus, officiels | Bibliothèques tierces ou méta-framework (Next.js…) | Bibliothèques officielles (Vue Router, Pinia) | Via SvelteKit |
| TypeScript | Obligatoire | Optionnel | Optionnel | Optionnel |
| Prise en main | La plus exigeante au départ | Moyenne | Douce | Douce |
| Terrain de prédilection | Grosses applications métier, équipes nombreuses, projets longs | Écosystème le plus vaste, très demandé à l'embauche | Adoption progressive, projets de toute taille | Performance, légèreté |

Ces frameworks **convergent** : tous reposent sur les composants, et la plupart ont adopté une réactivité par signaux ou par compilation. Les notions apprises avec Angular (composants, état, réactivité, services, tests) se transfèrent. En France, Angular est très présent dans les grandes entreprises, les ESN et le secteur public.

---

## On fait ensemble

1. Ouvrez https://angular.dev/playground. Le fichier `src/main.ts` contient un composant `Playground` et la ligne `bootstrapApplication(Playground);`.
2. Sélectionnez **tout** le contenu de `src/main.ts` et remplacez-le par l'application `CoffeeCounter` ci-dessus (section 3), imports et dernière ligne compris.
3. L'aperçu se recharge.

Résultat : « Cafés bus : 0 » et « Niveau d'énergie : endormi ». Chaque clic sur « Boire un café » incrémente le compteur ; le niveau passe à « efficace » au premier café, puis à « survolté » au quatrième. « Remettre à zéro » revient à l'état initial.

`energy` n'est jamais mis à jour à la main : c'est un `computed`, recalculé quand `cups` change.

Si l'aperçu affiche `Cannot find name 'Playground'`, la dernière ligne n'a pas été remplacée : elle doit démarrer la nouvelle classe, `bootstrapApplication(CoffeeCounter);`.

---

## Vous faites

**Exercice 1 — Choisir un framework.** Pour chaque situation, choisissez un framework et justifiez en deux phrases :
1. un back-office bancaire développé par 25 développeurs pendant 5 ans ;
2. un widget léger intégré dans un site existant ;
3. une startup qui doit recruter vite ;
4. le site vitrine d'une boulangerie.

<details>
<summary>Éléments de correction</summary>

1. **Angular** : cadre imposé et homogène, outillage complet officiel, TypeScript obligatoire, maintenance longue facilitée par `ng update`.
2. **Svelte ou Vue** : faible poids livré, intégration progressive dans l'existant.
3. **React** : vivier de développeurs le plus large. Angular reste un bon choix si l'équipe le maîtrise déjà.
4. **Aucun framework** : HTML et CSS, ou un générateur de site statique.

Il n'y a pas de réponse unique : ce qui compte, ce sont les critères (taille d'équipe, durée de vie, performance, recrutement).
</details>

**Exercice 2 — Lire du code.** Dans `CoffeeCounter`, repérez ce qui relève de l'état, du dérivé, de l'affichage et du comportement.

<details>
<summary>Correction</summary>

- État : `cups` (`signal`).
- Dérivé : `energy` (`computed`).
- Affichage : le `template` (interpolations `{{ }}`).
- Comportement : `drink()` et `reset()`, reliés aux boutons par `(click)`.
</details>

**Exercice 3 — Modifier.** Dans le playground, ajoutez un bouton « Boire deux cafés » et un paragraphe qui affiche « Pause obligatoire » à partir de 6 cafés. La méthode `drink` reçoit le nombre de cafés en paramètre.

<details>
<summary>Correction</summary>

```ts
import { Component, computed, signal } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';

@Component({
  selector: 'app-root',
  template: `
    <p>Cafés bus : {{ cups() }}</p>
    <p>Niveau d'énergie : {{ energy() }}</p>
    @if (cups() >= 6) {
      <p>Pause obligatoire</p>
    }
    <button type="button" (click)="drink(1)">Boire un café</button>
    <button type="button" (click)="drink(2)">Boire deux cafés</button>
    <button type="button" (click)="reset()">Remettre à zéro</button>
  `,
})
export class CoffeeCounter {
  protected readonly cups = signal(0);
  protected readonly energy = computed(() =>
    this.cups() === 0 ? 'endormi' : this.cups() < 4 ? 'efficace' : 'survolté',
  );

  protected drink(count: number): void {
    this.cups.update((cups) => cups + count);
  }

  protected reset(): void {
    this.cups.set(0);
  }
}

bootstrapApplication(CoffeeCounter);
```
</details>

---

## Check-list

- [ ] Je sais expliquer l'inversion de contrôle.
- [ ] Je sais citer trois principes d'Angular.
- [ ] Je sais justifier le choix d'Angular, ou d'un autre framework, pour un projet donné.

---

[← Sommaire](README.md) · [02 · TypeScript →](02-typescript.md)
