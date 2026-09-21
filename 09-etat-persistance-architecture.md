[← 08 · Routing](08-routing.md) · [Sommaire](README.md)

---

# Phase 08 — État, persistance et architecture

## Objectifs

- Choisir où placer un état : composant, service, URL, navigateur.
- Utiliser `effect` pour synchroniser un signal avec l'extérieur.
- Persister un état dans le `localStorage` de manière défensive.
- Organiser le code en couches et isoler la logique métier dans des fonctions pures.

Point d'arrivée : l'équipe et les devs créés sont conservés après rechargement ; la page « Mon équipe » affiche la couverture des types et les moyennes.

---

## On théorise

### 1. Où placer un état ?

| État | Emplacement | Exemple dans Pokedev |
|---|---|---|
| Propre à un composant | `signal` dans le composant | la statistique mise en avant sur une fiche |
| Partagé entre composants | `signal` dans un service | l'équipe |
| Partageable par lien | l'URL (paramètres) | le filtre par type, le numéro du dev |
| Conservé après rechargement | `localStorage` + service | l'équipe, les devs créés |
| Venant d'un serveur | `httpResource` dans un service | la liste des devs |

Tout ce qui peut être **calculé** à partir d'un autre état est un `computed`, jamais une copie tenue à jour à la main.

### 2. `effect`

Un `effect` exécute une fonction chaque fois qu'un signal qu'elle lit change. Il sert à **synchroniser un signal avec l'extérieur** : `localStorage`, titre de page, bibliothèque non Angular.

```ts
effect(() => {
  localStorage.setItem(key, JSON.stringify(state()));
});
```

- Un `effect` s'exécute de façon asynchrone, après la modification.
- On ne l'utilise **pas** pour calculer un état à partir d'un autre : c'est le rôle de `computed`.
- Comme `inject()`, il se crée dans un contexte d'injection.

### 3. Le `localStorage` n'est pas fiable

Son contenu peut être absent, modifié à la main, corrompu, ou écrit par une ancienne version de l'application. L'accès peut échouer (navigation privée, quota). On applique donc la même règle qu'aux réponses HTTP : **valider à la frontière**.
- lecture dans un `try` / `catch`, avec une valeur par défaut ;
- vérification de la forme avec un type guard ;
- clé versionnée (`pokedev.team.v1`) pour pouvoir changer de format plus tard.

### 4. Architecture en couches

```
src/app/
├── domain/     modèle et règles métier ── fonctions pures, AUCUN import Angular
├── core/       services transverses ───── données, HTTP, navigation, stockage, équipe
├── shared/     briques d'interface ────── composants, directives, pipes réutilisables
└── features/   une page par dossier ───── orchestre core et shared
```

Règle de dépendance : `features` → `shared`, `core` → `domain`. Jamais l'inverse : `domain` ne connaît ni Angular, ni HTTP, ni le DOM.

Une **fonction pure** renvoie toujours le même résultat pour les mêmes arguments et ne modifie rien à l'extérieur. Elle est testable sans navigateur ni Angular, et réutilisable dans un autre framework ou côté serveur.

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 08.

### Étape 1 — Un signal persisté

Créez `src/app/core/storage/persisted-signal.ts` :

`src/app/core/storage/persisted-signal.ts`

```ts
import { WritableSignal, effect, signal } from '@angular/core';

/**
 * Signal dont la valeur est sauvegardée dans le localStorage à chaque modification.
 * `isValid` vérifie la valeur relue : le contenu du localStorage n'est pas fiable.
 * À appeler dans un contexte d'injection (champ ou constructeur d'un service).
 */
export function persistedSignal<T>(
  key: string,
  initial: T,
  isValid: (value: unknown) => value is T,
): WritableSignal<T> {
  const state = signal<T>(read(key, initial, isValid));
  effect(() => {
    const value = state();
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // quota dépassé ou stockage indisponible : l'application reste utilisable
    }
  });
  return state;
}

function read<T>(key: string, initial: T, isValid: (value: unknown) => value is T): T {
  try {
    const raw = localStorage.getItem(key);
    if (raw !== null) {
      const parsed: unknown = JSON.parse(raw);
      if (isValid(parsed)) {
        return parsed;
      }
    }
  } catch {
    // JSON corrompu ou stockage indisponible
  }
  return initial;
}
```

La fonction crée un `signal`, le remplit à partir du `localStorage`, et installe un `effect` qui sauvegarde chaque modification.

### Étape 2 — L'équipe persistée

Remplacez `src/app/core/team/team.ts` :

`src/app/core/team/team.ts`

```ts
import { Service, computed, inject } from '@angular/core';
import { MAX_TEAM_SIZE } from '../../domain/dev.model';
import { averageStats, teamCoverage } from '../../domain/dev-rules';
import { DevRepository } from '../data/dev-repository';
import { persistedSignal } from '../storage/persisted-signal';

const TEAM_KEY = 'pokedev.team.v1';

function isIdList(value: unknown): value is number[] {
  return Array.isArray(value) && value.every((id) => Number.isInteger(id));
}

/** L'équipe de l'utilisateur : au plus six devs, conservée dans le navigateur. */
@Service()
export class Team {
  private readonly repository = inject(DevRepository);
  private readonly ids = persistedSignal<number[]>(TEAM_KEY, [], isIdList);

  readonly members = computed(() =>
    this.ids()
      .map((id) => this.repository.byId(id))
      .filter((dev) => dev !== undefined),
  );
  readonly size = computed(() => this.ids().length);
  readonly isFull = computed(() => this.size() >= MAX_TEAM_SIZE);
  readonly coverage = computed(() => teamCoverage(this.members()));
  readonly average = computed(() => averageStats(this.members()));

  has(id: number): boolean {
    return this.ids().includes(id);
  }

  /** Ajoute ou retire un dev. Renvoie false si l'équipe est pleine. */
  toggle(id: number): boolean {
    if (this.has(id)) {
      this.ids.update((ids) => ids.filter((current) => current !== id));
      return true;
    }
    if (this.isFull()) {
      return false;
    }
    this.ids.update((ids) => [...ids, id]);
    return true;
  }

  clear(): void {
    this.ids.set([]);
  }
}
```

Nouveautés : l'état est un `persistedSignal` ; `members`, `coverage` et `average` sont des `computed` qui s'appuient sur les fonctions pures du domaine.

### Étape 3 — Les devs créés par l'utilisateur

Le dépôt fusionne les devs du fichier JSON et ceux créés par l'utilisateur, conservés dans le navigateur. Il prépare le formulaire de la phase 10. Remplacez `src/app/core/data/dev-repository.ts` :

`src/app/core/data/dev-repository.ts`

```ts
import { Service, computed, inject } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { Dev } from '../../domain/dev.model';
import { nextId } from '../../domain/dev-rules';
import { persistedSignal } from '../storage/persisted-signal';
import { isDev, parseDevs } from './dev-validation';
import { DEVS_URL } from './devs-url';

const CUSTOM_DEVS_KEY = 'pokedev.custom-devs.v1';

function isDevList(value: unknown): value is Dev[] {
  return Array.isArray(value) && value.every(isDev);
}

/** Source unique des devs : ceux du fichier JSON et ceux créés par l'utilisateur. */
@Service()
export class DevRepository {
  private readonly url = inject(DEVS_URL);
  private readonly remote = httpResource(() => this.url, {
    parse: parseDevs,
    defaultValue: [],
  });
  private readonly custom = persistedSignal<Dev[]>(CUSTOM_DEVS_KEY, [], isDevList);

  /**
   * Lire value() d'une ressource en erreur lève une exception :
   * on vérifie hasValue() avant, pour garder les devs personnalisés affichables.
   */
  readonly devs = computed(() => [
    ...(this.remote.hasValue() ? this.remote.value() : []),
    ...this.custom(),
  ]);
  readonly isLoading = this.remote.isLoading;
  readonly error = this.remote.error;

  byId(id: number): Dev | undefined {
    return this.devs().find((dev) => dev.id === id);
  }

  /** Ajoute un dev créé par l'utilisateur et renvoie son numéro. */
  add(dev: Omit<Dev, 'id' | 'custom'>): number {
    const id = nextId(this.devs());
    this.custom.update((devs) => [...devs, { ...dev, id, custom: true }]);
    return id;
  }

  reload(): void {
    this.remote.reload();
  }
}
```

`add` attribue le numéro suivant avec `nextId`, une fonction pure du domaine.

### Résultat attendu

1. Ajoutez Stagiairon et Requêtor à l'équipe : le badge de l'en-tête affiche 2.
2. Dans les outils de développement, onglet « Application », section « Stockage local » : la clé `pokedev.team.v1` vaut `[1,11]`.
3. Rechargez la page : le badge affiche toujours 2.
4. Double-cliquez sur la valeur de la clé, remplacez-la par `{pas du json` et rechargez : l'application démarre avec une équipe vide, sans erreur dans la console.

---

## Vous faites

**Exercice 1 — Relire les règles de l'équipe.** Dans `src/app/domain/dev-rules.ts`, relisez `teamCoverage`, `averageStats` et `previousEvolution`, écrites en phase 05. Pour chacune, prédisez le résultat pour une équipe composée de Tabulis (Data) et Requêtor (Data, Back-end), puis vérifiez à l'exercice 2.

<details>
<summary>Correction</summary>

- `teamCoverage` : `['backend', 'data']`, dans l'ordre de `DEV_TYPES` : 2 types couverts sur 6.
- `averageStats` : la moyenne arrondie de chaque statistique ; le total de cette moyenne vaut 289.
- `previousEvolution(devs, requêtor)` : Tabulis, qui évolue vers Requêtor.
</details>

**Exercice 2 — La page « Mon équipe ».** Créez `src/app/features/team/team-page.ts` (avec `.html` et `.css`) et la route `equipe`, de titre « Pokedev · Mon équipe », placée avant la route `**`. La page affiche :
- le titre « Mon équipe (n/6) » ;
- un état vide avec un lien vers le pokédex si l'équipe est vide ;
- les membres : avatar, numéro et nom avec un lien vers la fiche, bouton « Retirer » ;
- le nombre de types couverts et la liste des types manquants, séparés par des virgules, avec un point final ;
- la moyenne de chaque statistique (`StatBar`) et le total de cette moyenne, formaté avec le pipe `number` ;
- un bouton « Vider l'équipe ».

<details>
<summary>Correction</summary>

`src/app/features/team/team-page.ts`

```ts
import { Component, computed, inject } from '@angular/core';
import { DecimalPipe } from '@angular/common';
import { RouterLink } from '@angular/router';
import { Team } from '../../core/team/team';
import { DEV_TYPES, MAX_TEAM_SIZE, STAT_KEYS } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';
import { STAT_LABELS } from '../../domain/labels';
import { DexNumberPipe } from '../../shared/pipes/dex-number-pipe';
import { TypeLabelPipe } from '../../shared/pipes/type-label-pipe';
import { DevAvatar } from '../../shared/ui/dev-avatar';
import { EmptyState } from '../../shared/ui/empty-state';
import { StatBar } from '../../shared/ui/stat-bar';

@Component({
  selector: 'app-team-page',
  imports: [DecimalPipe, RouterLink, DexNumberPipe, TypeLabelPipe, DevAvatar, EmptyState, StatBar],
  templateUrl: './team-page.html',
  styleUrl: './team-page.css',
})
export class TeamPage {
  protected readonly team = inject(Team);

  protected readonly maxSize = MAX_TEAM_SIZE;
  protected readonly statKeys = STAT_KEYS;
  protected readonly statLabels = STAT_LABELS;

  protected readonly missingTypes = computed(() =>
    DEV_TYPES.filter((type) => !this.team.coverage().includes(type)),
  );
  protected readonly averageTotal = computed(() => {
    const average = this.team.average();
    return average ? totalStats(average) : 0;
  });
}
```

`src/app/features/team/team-page.html`

```html
<h1>Mon équipe ({{ team.size() }}/{{ maxSize }})</h1>

@if (team.members().length === 0) {
  <app-empty-state>
    <p>Votre équipe est vide.</p>
    <a actions routerLink="/devs">Choisir des devs dans le pokédex</a>
  </app-empty-state>
} @else {
  <ul class="members">
    @for (dev of team.members(); track dev.id) {
      <li>
        <app-dev-avatar [name]="dev.name" [type]="dev.types[0]" />
        <a [routerLink]="['/devs', dev.id]">{{ dev.id | dexNumber }} {{ dev.name }}</a>
        <button type="button" (click)="team.toggle(dev.id)">Retirer</button>
      </li>
    }
  </ul>

  <section aria-labelledby="coverage-title">
    <h2 id="coverage-title">Couverture : {{ team.coverage().length }} type(s) sur 6</h2>
    @if (missingTypes().length > 0) {
      <p>
        Il manque :
        @for (type of missingTypes(); track type; let last = $last) {
          {{ type | typeLabel }}{{ last ? '.' : ', ' }}
        }
      </p>
    } @else {
      <p>Tous les types sont couverts.</p>
    }
  </section>

  @if (team.average(); as average) {
    <section aria-labelledby="average-title">
      <h2 id="average-title">Moyenne de l'équipe · total {{ averageTotal() | number }}</h2>
      <div class="stats">
        @for (key of statKeys; track key) {
          <app-stat-bar [label]="statLabels[key]" [value]="average[key]" />
        }
      </div>
    </section>
  }

  <p><button type="button" (click)="team.clear()">Vider l'équipe</button></p>
}
```

`src/app/features/team/team-page.css`

```css
.members {
  display: grid;
  gap: 0.75rem;
  padding: 0;
  list-style: none;
}
.members li {
  display: grid;
  grid-template-columns: auto 1fr auto;
  align-items: center;
  gap: 1rem;
}
.stats {
  display: grid;
  gap: 0.4rem;
  max-width: 32rem;
}
```

`src/app/app.routes.ts`

```ts
import { Routes } from '@angular/router';
import { devIdGuard } from './core/navigation/dev-id-guard';
import { devTitleResolver } from './core/navigation/dev-title-resolver';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'devs' },
  {
    path: 'devs',
    loadComponent: () => import('./features/dex/dex-page').then((m) => m.DexPage),
    title: 'Pokedev · Pokédex',
  },
  {
    path: 'devs/:id',
    loadComponent: () =>
      import('./features/detail/dev-detail-page').then((m) => m.DevDetailPage),
    canActivate: [devIdGuard],
    title: devTitleResolver,
  },
  {
    path: 'equipe',
    loadComponent: () => import('./features/team/team-page').then((m) => m.TeamPage),
    title: 'Pokedev · Mon équipe',
  },
  {
    path: '**',
    loadComponent: () =>
      import('./features/not-found/not-found-page').then((m) => m.NotFoundPage),
    title: 'Pokedev · Page introuvable',
  },
];
```

Résultat : avec Tabulis et Requêtor, « Mon équipe (2/6) », « Couverture : 2 type(s) sur 6 », « Il manque : Front-end, DevOps, Mobile, Sécurité. » et « Moyenne de l'équipe · total 289 ».
</details>

**Exercice 3 — Vérifier l'architecture.**
1. Lancez `grep -rn "@angular" src/app/domain` (sous Windows PowerShell : `Select-String -Path src/app/domain/*.ts -Pattern "@angular"`). La commande ne doit rien afficher.
2. Pour chacun des fichiers suivants, indiquez la couche et justifiez : `persisted-signal.ts`, `dev-avatar.ts`, `labels.ts`, `loading-interceptor.ts`, `team-page.ts`.
3. Un collègue propose d'appeler `localStorage` directement dans `DevCard` pour savoir si un dev est dans l'équipe. Expliquez pourquoi c'est une mauvaise idée.

<details>
<summary>Correction</summary>

2. `persisted-signal.ts` : `core` (technique, transverse) ; `dev-avatar.ts` : `shared` (interface réutilisable) ; `labels.ts` : `domain` (données métier, sans Angular) ; `loading-interceptor.ts` : `core` ; `team-page.ts` : `features`.
3. La carte deviendrait dépendante du stockage : elle ne serait plus réutilisable ni testable sans navigateur, la lecture défensive serait dupliquée, et l'affichage ne se mettrait pas à jour quand l'équipe change ailleurs (le `localStorage` n'est pas un signal). La carte doit recevoir `inTeam` en entrée.
</details>

---

## Check-list

- [ ] Je sais choisir où placer un état.
- [ ] Je sais utiliser `effect` pour synchroniser un signal avec l'extérieur, et pas pour calculer un état.
- [ ] Je lis le `localStorage` de manière défensive.
- [ ] Je sais expliquer la règle de dépendance entre `domain`, `core`, `shared` et `features`.

---

[← 08 · Routing](08-routing.md) · [Sommaire](README.md) · [10 · Formulaires →](10-formulaires.md)
