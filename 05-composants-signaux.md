[← 04 · Conception](04-conception.md) · [Sommaire](README.md)

---

# Phase 04 — Composants, templates et signaux

## Objectifs

- Écrire un composant : décorateur, template, styles, imports.
- Utiliser les liaisons de template : interpolation, propriété, attribut, classe, style, événement.
- Utiliser le control flow : `@if`, `@for`, `@switch`, `@let`.
- Gérer l'état avec `signal` et `computed`.
- Faire communiquer des composants avec `input()` et `output()`.
- Projeter du contenu avec `<ng-content>`.

Point d'arrivée : la liste des devs, alimentée par des données fictives, avec ajout et retrait de l'équipe.

---

## On théorise

### 1. Anatomie d'un composant

```ts
@Component({
  selector: 'app-dev-card',     // la balise : <app-dev-card />
  imports: [DevAvatar],         // les composants, directives et pipes utilisés dans le template
  templateUrl: './dev-card.html', // ou template: `...`
  styleUrl: './dev-card.css',     // ou styles: `...`
})
export class DevCard {
  // état, entrées, sorties, méthodes
}
```

Les styles d'un composant sont **encapsulés** : `.card` dans `dev-card.css` ne touche que ce composant. `:host` désigne l'élément du composant lui-même.

### 2. Les liaisons de template

| Syntaxe | Rôle | Exemple |
|---|---|---|
| `{{ expr }}` | Interpolation (texte échappé) | `{{ dev().name }}` |
| `[prop]="expr"` | Propriété d'un élément ou entrée d'un composant | `[disabled]="teamFull()"`, `[dev]="dev"` |
| `[attr.x]="expr"` | Attribut HTML (ARIA…) | `[attr.aria-pressed]="inTeam()"` |
| `[class.x]="bool"` | Ajoute ou retire une classe | `[class.in-team]="inTeam()"` |
| `[style.x]="expr"` | Style en ligne, avec unité possible | `[style.width.%]="percent()"` |
| `(event)="instruction"` | Événement | `(click)="toggle(dev.id)"` |
| `#ref` | Référence locale vers un élément | `<input #search />` |

### 3. Le control flow

```html
@if (devs().length > 0) {
  <p>{{ devs().length }} dev(s)</p>
} @else {
  <p>Aucun dev.</p>
}

@for (dev of devs(); track dev.id) {
  <app-dev-card [dev]="dev" />
} @empty {
  <p>Aucun dev.</p>
}

@switch (rank()) {
  @case ('junior') { <span>Junior</span> }
  @default { <span>Senior</span> }
}

@let current = dev();  <!-- variable locale au template -->
```

`track` est obligatoire dans `@for` : il identifie chaque élément pour qu'Angular ne recrée que ce qui a changé. `@for` fournit aussi `$index`, `$first`, `$last`, `$count`.

### 4. Les signaux

```ts
const cups = signal(0);         // état modifiable
cups();                         // lecture : on appelle le signal
cups.set(3);                    // remplacement
cups.update((n) => n + 1);      // calcul à partir de l'ancienne valeur

const energy = computed(() => (cups() > 3 ? 'survolté' : 'efficace')); // dérivé, en lecture seule
```

- Un `computed` se recalcule **seulement** quand un signal qu'il lit change, et seulement si on le lit.
- Quand un signal lu dans un template change, Angular met à jour **ce composant**. C'est ce qui permet à Angular 22 de fonctionner sans zone.js.
- On ne modifie jamais un tableau ou un objet en place (`push`) : on crée une nouvelle valeur (`[...ids, id]`). Sinon, le signal ne voit pas de changement.

### 5. Entrées et sorties

```
   DexPage (parent)                                     DevCard (enfant)
   ─────────────────                                    ────────────────
   [dev]="dev"              ───── input ─────▶          readonly dev = input.required<Dev>()
   (teamToggled)="toggle($event)" ◀── output ───        readonly teamToggled = output<number>()
```

- `input()` crée un signal en lecture seule alimenté par le parent. `input.required<T>()` rend l'entrée obligatoire ; `input(false)` fournit une valeur par défaut.
- `output<T>()` déclare un événement ; l'enfant appelle `emit(valeur)`, le parent reçoit la valeur dans `$event`.
- Les données descendent, les événements remontent : l'enfant ne modifie jamais directement l'état du parent.

### 6. Projection de contenu

Un composant peut recevoir du HTML de son parent :

```html
<!-- empty-state.ts : template -->
<div class="message"><ng-content /></div>
<div class="actions"><ng-content select="[actions]" /></div>

<!-- utilisation -->
<app-empty-state>
  <p>Aucun dev ne correspond.</p>
  <button actions type="button">Effacer</button>
</app-empty-state>
```

---

## On fait ensemble

Point de départ : le projet de la phase 03. Laissez `npx ng serve` tourner : chaque enregistrement met la page à jour.

### Étape 1 — Le domaine

Créez le dossier `src/app/domain/` et trois fichiers. C'est le code de la phase 02, réparti en fichiers : chaque élément utilisé ailleurs est précédé de `export`.

`src/app/domain/dev.model.ts`

```ts
/** Les six types de développeur, équivalents des types élémentaires d'un pokédex. */
export const DEV_TYPES = ['frontend', 'backend', 'devops', 'data', 'mobile', 'securite'] as const;
export type DevType = (typeof DEV_TYPES)[number];

/** Les six statistiques d'un dev, chacune entre 0 et MAX_STAT. */
export const STAT_KEYS = ['code', 'debug', 'archi', 'tests', 'communication', 'cafe'] as const;
export type StatKey = (typeof STAT_KEYS)[number];
export type DevStats = Record<StatKey, number>;

export const MAX_STAT = 100;
export const MAX_TOTAL = 420;
export const MAX_TEAM_SIZE = 6;

export interface Dev {
  readonly id: number;
  readonly name: string;
  readonly title: string;
  /** Un ou deux types, le premier est le type principal. */
  readonly types: readonly DevType[];
  readonly stats: DevStats;
  readonly languages: readonly string[];
  readonly catchphrase: string;
  /** Numéro du dev vers lequel celui-ci évolue. */
  readonly evolvesTo?: number;
  /** Vrai pour un dev créé par l'utilisateur. */
  readonly custom?: boolean;
}

export interface DevFilter {
  readonly query: string;
  readonly type?: DevType;
}
```

`src/app/domain/dev-rules.ts`

```ts
import { DEV_TYPES, Dev, DevFilter, DevStats, DevType, STAT_KEYS, StatKey } from './dev.model';

/** Somme des six statistiques. */
export function totalStats(stats: DevStats): number {
  return STAT_KEYS.reduce((sum, key) => sum + stats[key], 0);
}

/** Statistique la plus élevée ; en cas d'égalité, la première dans l'ordre de STAT_KEYS. */
export function bestStat(stats: DevStats): StatKey {
  return STAT_KEYS.reduce((best, key) => (stats[key] > stats[best] ? key : best));
}

/** Numéro affiché façon pokédex : 7 → « #007 ». */
export function formatDexNumber(id: number): string {
  return `#${String(id).padStart(3, '0')}`;
}

/** Normalise une chaîne pour une recherche insensible à la casse et aux accents. */
function normalize(text: string): string {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().trim();
}

export function matchesFilter(dev: Dev, filter: DevFilter): boolean {
  if (filter.type && !dev.types.includes(filter.type)) {
    return false;
  }
  const query = normalize(filter.query);
  if (query === '') {
    return true;
  }
  return [dev.name, dev.title, ...dev.languages].some((text) => normalize(text).includes(query));
}

export function filterDevs(devs: readonly Dev[], filter: DevFilter): Dev[] {
  return devs.filter((dev) => matchesFilter(dev, filter));
}

/** Types représentés dans une équipe, dans l'ordre de DEV_TYPES. */
export function teamCoverage(devs: readonly Dev[]): DevType[] {
  return DEV_TYPES.filter((type) => devs.some((dev) => dev.types.includes(type)));
}

/** Moyenne arrondie de chaque statistique ; null pour une équipe vide. */
export function averageStats(devs: readonly Dev[]): DevStats | null {
  if (devs.length === 0) {
    return null;
  }
  const average = {} as DevStats;
  for (const key of STAT_KEYS) {
    average[key] = Math.round(devs.reduce((sum, dev) => sum + dev.stats[key], 0) / devs.length);
  }
  return average;
}

/** Numéro libre suivant : le plus grand numéro existant + 1. */
export function nextId(devs: readonly Dev[]): number {
  return devs.reduce((max, dev) => Math.max(max, dev.id), 0) + 1;
}

/** Le dev qui évolue vers `dev`, s'il existe. */
export function previousEvolution(devs: readonly Dev[], dev: Dev): Dev | undefined {
  return devs.find((candidate) => candidate.evolvesTo === dev.id);
}
```

Les fonctions `teamCoverage`, `averageStats` et `previousEvolution` serviront à la phase 09.

`src/app/domain/labels.ts`

```ts
import { DevType, StatKey } from './dev.model';

export const TYPE_LABELS: Record<DevType, string> = {
  frontend: 'Front-end',
  backend: 'Back-end',
  devops: 'DevOps',
  data: 'Data',
  mobile: 'Mobile',
  securite: 'Sécurité',
};

export const STAT_LABELS: Record<StatKey, string> = {
  code: 'Code',
  debug: 'Debug',
  archi: 'Architecture',
  tests: 'Tests',
  communication: 'Communication',
  cafe: 'Café',
};
```

### Étape 2 — Des données fictives

Créez `src/app/features/dex/mock-devs.ts` :

`src/app/features/dex/mock-devs.ts`

```ts
import { Dev } from '../../domain/dev.model';

/** Données fictives, en attendant le chargement HTTP. */
export const MOCK_DEVS: Dev[] = [
  {
    id: 1,
    name: 'Stagiairon',
    title: 'Stagiaire front-end',
    types: ['frontend'],
    stats: { code: 35, debug: 20, archi: 10, tests: 15, communication: 40, cafe: 60 },
    languages: ['HTML', 'CSS'],
    catchphrase: 'Ça marche sur ma machine.',
    evolvesTo: 2,
  },
  {
    id: 8,
    name: 'Kubernaute',
    title: 'Ingénieur DevOps',
    types: ['devops'],
    stats: { code: 55, debug: 60, archi: 50, tests: 40, communication: 40, cafe: 65 },
    languages: ['YAML', 'Go', 'Bash'],
    catchphrase: 'Redémarre le pod.',
  },
  {
    id: 11,
    name: 'Requêtor',
    title: 'Ingénieur data',
    types: ['data', 'backend'],
    stats: { code: 65, debug: 60, archi: 60, tests: 50, communication: 45, cafe: 55 },
    languages: ['SQL', 'Python', 'Scala'],
    catchphrase: 'Ajoute un index.',
  },
];
```

### Étape 3 — La carte d'un dev

Créez `src/app/features/dex/dev-card.ts` (ou générez-le avec `npx ng g component features/dex/dev-card --flat --inline-template --inline-style --skip-tests`, puis remplacez son contenu) :

`src/app/features/dex/dev-card.ts`

```ts
import { Component, computed, input, output } from '@angular/core';
import { Dev } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';

@Component({
  selector: 'app-dev-card',
  template: `
    <article class="card" [class.in-team]="inTeam()">
      <p>#{{ dev().id }}</p>
      <h3>{{ dev().name }}</h3>
      <p>{{ dev().title }}</p>
      <p>
        @for (type of dev().types; track type) {
          <span class="type">{{ type }}</span>
        }
      </p>
      <p>Total : {{ total() }}</p>
      <button type="button" (click)="teamToggled.emit(dev().id)">
        {{ inTeam() ? 'Retirer de l’équipe' : 'Ajouter à l’équipe' }}
      </button>
    </article>
  `,
  styles: `
    .card {
      padding: 1rem;
      border: 1px solid var(--border);
      border-radius: 0.75rem;
    }
    .card.in-team {
      border-color: var(--accent);
      box-shadow: 0 0 0 1px var(--accent);
    }
    .type {
      margin-right: 0.25rem;
      padding: 0 0.5rem;
      border: 1px solid var(--border);
      border-radius: 999px;
    }
  `,
})
export class DevCard {
  readonly dev = input.required<Dev>();
  readonly inTeam = input(false);
  readonly teamToggled = output<number>();

  protected readonly total = computed(() => totalStats(this.dev().stats));
}
```

- `dev` est **obligatoire** ; `inTeam` vaut `false` par défaut.
- `total` est un `computed` : il se recalcule si l'entrée `dev` change.
- La carte ne gère pas l'équipe : elle **émet** le numéro du dev, le parent décide.

### Étape 4 — La liste

Créez `src/app/features/dex/dex-page.ts` :

`src/app/features/dex/dex-page.ts`

```ts
import { Component, signal } from '@angular/core';
import { DevCard } from './dev-card';
import { MOCK_DEVS } from './mock-devs';

@Component({
  selector: 'app-dex-page',
  imports: [DevCard],
  template: `
    <h1>Pokédex</h1>
    <p>{{ devs().length }} dev(s) · {{ teamIds().length }} dans l'équipe</p>
    <ul class="grid">
      @for (dev of devs(); track dev.id) {
        <li>
          <app-dev-card
            [dev]="dev"
            [inTeam]="teamIds().includes(dev.id)"
            (teamToggled)="toggle($event)"
          />
        </li>
      } @empty {
        <li>Aucun dev.</li>
      }
    </ul>
  `,
  styles: `
    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr));
      gap: 1rem;
      padding: 0;
      list-style: none;
    }
  `,
})
export class DexPage {
  protected readonly devs = signal(MOCK_DEVS);
  protected readonly teamIds = signal<number[]>([]);

  protected toggle(id: number): void {
    this.teamIds.update((ids) =>
      ids.includes(id) ? ids.filter((current) => current !== id) : [...ids, id],
    );
  }
}
```

### Étape 5 — Afficher la liste

Remplacez `src/app/app.ts` et `src/app/app.html` :

`src/app/app.ts`

```ts
import { Component } from '@angular/core';
import { DexPage } from './features/dex/dex-page';

@Component({
  selector: 'app-root',
  imports: [DexPage],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {}
```

`src/app/app.html`

```html
<header>
  <strong class="brand">Pokedev</strong>
</header>
<main>
  <app-dex-page />
</main>
```

Résultat attendu :
- trois cartes : Stagiairon, Kubernaute, Requêtor, avec « 3 dev(s) · 0 dans l'équipe » ;
- un clic sur « Ajouter à l'équipe » : le compteur passe à 1, le bouton devient « Retirer de l'équipe » et la carte prend une bordure bleue ;
- un second clic annule.

Ouvrez Angular DevTools (onglet « Angular » des outils de développement), sélectionnez `DexPage` et observez la valeur de `teamIds` pendant les clics.

Le test de `app.spec.ts` passe toujours (`npx ng test`) : il cherche le texte « Pokedev », présent dans l'en-tête.

---

## Vous faites

**Exercice 1 — Une barre de statistique.** Créez `src/app/shared/ui/stat-bar.ts`, un composant `StatBar` :
- entrées : `label` (obligatoire), `value` (obligatoire), `max` (défaut `MAX_STAT`), `highlight` (défaut `false`) ;
- affichage : le libellé, une barre dont la largeur est proportionnelle à la valeur, la valeur ;
- le libellé est en gras quand `highlight` vaut `true` ;
- pour l'accessibilité, l'élément hôte porte `role="meter"`, `aria-label`, `aria-valuenow`, `aria-valuemin` et `aria-valuemax`.

Affichez les six statistiques du dev dans la carte.

Indice : les liaisons sur l'élément hôte se déclarent dans la propriété `host` du décorateur, par exemple `host: { '[class.highlight]': 'highlight()' }`.

<details>
<summary>Correction</summary>

`src/app/shared/ui/stat-bar.ts`

```ts
import { Component, computed, input } from '@angular/core';
import { MAX_STAT } from '../../domain/dev.model';

/** Barre horizontale représentant une statistique sur 100. */
@Component({
  selector: 'app-stat-bar',
  template: `
    <span class="label">{{ label() }}</span>
    <span class="track" aria-hidden="true">
      <span class="fill" [style.width.%]="percent()"></span>
    </span>
    <span class="value">{{ value() }}</span>
  `,
  host: {
    role: 'meter',
    '[attr.aria-label]': 'label()',
    '[attr.aria-valuenow]': 'value()',
    'aria-valuemin': '0',
    '[attr.aria-valuemax]': 'max()',
    '[class.highlight]': 'highlight()',
  },
  styles: `
    :host {
      display: grid;
      grid-template-columns: 8rem 1fr 2.5rem;
      align-items: center;
      gap: 0.5rem;
    }
    .track {
      height: 0.6rem;
      border-radius: 999px;
      background: var(--border);
      overflow: hidden;
    }
    .fill {
      display: block;
      height: 100%;
      background: var(--type-color, var(--accent));
    }
    .value {
      text-align: end;
      font-variant-numeric: tabular-nums;
    }
    :host(.highlight) .label {
      font-weight: 700;
    }
  `,
})
export class StatBar {
  readonly label = input.required<string>();
  readonly value = input.required<number>();
  readonly max = input(MAX_STAT);
  readonly highlight = input(false);

  protected readonly percent = computed(() =>
    Math.min(100, Math.round((this.value() / this.max()) * 100)),
  );
}
```

La carte, avec les barres et le niveau de l'exercice 3 :

`src/app/features/dex/dev-card.ts`

```ts
import { Component, computed, input, output } from '@angular/core';
import { Dev, STAT_KEYS } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';
import { STAT_LABELS } from '../../domain/labels';
import { StatBar } from '../../shared/ui/stat-bar';
import { RankLabel } from './rank-label';

@Component({
  selector: 'app-dev-card',
  imports: [StatBar, RankLabel],
  template: `
    <article class="card" [class.in-team]="inTeam()">
      <p>#{{ dev().id }}</p>
      <h3>{{ dev().name }}</h3>
      <p>{{ dev().title }}</p>
      <p>
        @for (type of dev().types; track type) {
          <span class="type">{{ type }}</span>
        }
      </p>
      <p>Total : {{ total() }} · <app-rank-label [total]="total()" /></p>
      @for (key of statKeys; track key) {
        <app-stat-bar [label]="statLabels[key]" [value]="dev().stats[key]" />
      }
      <button type="button" (click)="teamToggled.emit(dev().id)">
        {{ inTeam() ? 'Retirer de l’équipe' : 'Ajouter à l’équipe' }}
      </button>
    </article>
  `,
  styles: `
    .card {
      padding: 1rem;
      border: 1px solid var(--border);
      border-radius: 0.75rem;
    }
    .card.in-team {
      border-color: var(--accent);
      box-shadow: 0 0 0 1px var(--accent);
    }
    .type {
      margin-right: 0.25rem;
      padding: 0 0.5rem;
      border: 1px solid var(--border);
      border-radius: 999px;
    }
  `,
})
export class DevCard {
  readonly dev = input.required<Dev>();
  readonly inTeam = input(false);
  readonly teamToggled = output<number>();

  protected readonly statKeys = STAT_KEYS;
  protected readonly statLabels = STAT_LABELS;
  protected readonly total = computed(() => totalStats(this.dev().stats));
}
```

Résultat : six barres par carte ; celle du code de Stagiairon est remplie à 35 %.
</details>

**Exercice 2 — Un état vide réutilisable.** Créez `src/app/shared/ui/empty-state.ts`, un composant `EmptyState` qui affiche un cadre en pointillés contenant :
- le contenu projeté par défaut ;
- un emplacement `[actions]` pour des boutons ou des liens, masqué s'il est vide.

Dans la liste, remplacez `@empty` par un `@if` / `@else` qui affiche `EmptyState` avec un bouton « Recharger ». Pour tester, ajoutez au-dessus de la liste un bouton « Vider la liste » qui appelle `devs.set([])`.

<details>
<summary>Correction</summary>

`src/app/shared/ui/empty-state.ts`

```ts
import { Component } from '@angular/core';

/**
 * Bloc « état vide » : le contenu est projeté par le parent.
 * Le slot [actions] reçoit les boutons ou liens, le reste va dans le message.
 */
@Component({
  selector: 'app-empty-state',
  template: `
    <div class="message"><ng-content /></div>
    <div class="actions"><ng-content select="[actions]" /></div>
  `,
  styles: `
    :host {
      display: block;
      padding: 2rem 1rem;
      border: 2px dashed var(--border);
      border-radius: 0.75rem;
      text-align: center;
    }
    .actions:empty {
      display: none;
    }
    .actions {
      margin-top: 1rem;
    }
  `,
})
export class EmptyState {}
```

`.actions:empty` masque l'emplacement quand rien n'y est projeté.

`src/app/features/dex/dex-page.ts`

```ts
import { Component, signal } from '@angular/core';
import { EmptyState } from '../../shared/ui/empty-state';
import { DevCard } from './dev-card';
import { MOCK_DEVS } from './mock-devs';

@Component({
  selector: 'app-dex-page',
  imports: [DevCard, EmptyState],
  template: `
    <h1>Pokédex</h1>
    <p>{{ devs().length }} dev(s) · {{ teamIds().length }} dans l'équipe</p>
    <button type="button" (click)="devs.set([])">Vider la liste</button>

    @if (devs().length > 0) {
      <ul class="grid">
        @for (dev of devs(); track dev.id) {
          <li>
            <app-dev-card
              [dev]="dev"
              [inTeam]="teamIds().includes(dev.id)"
              (teamToggled)="toggle($event)"
            />
          </li>
        }
      </ul>
    } @else {
      <app-empty-state>
        <p>Aucun dev.</p>
        <button actions type="button" (click)="devs.set(MOCK_DEVS)">Recharger</button>
      </app-empty-state>
    }
  `,
  styles: `
    .grid {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr));
      gap: 1rem;
      padding: 0;
      list-style: none;
    }
  `,
})
export class DexPage {
  protected readonly MOCK_DEVS = MOCK_DEVS;
  protected readonly devs = signal(MOCK_DEVS);
  protected readonly teamIds = signal<number[]>([]);

  protected toggle(id: number): void {
    this.teamIds.update((ids) =>
      ids.includes(id) ? ids.filter((current) => current !== id) : [...ids, id],
    );
  }
}
```

`MOCK_DEVS` est exposé comme champ : un template n'a accès qu'aux membres de sa classe.

Résultat : « Vider la liste » affiche le cadre « Aucun dev. » et le bouton « Recharger », qui rétablit les trois cartes.
</details>

**Exercice 3 — Control flow.** Créez `src/app/features/dex/rank-label.ts`, un composant `RankLabel` qui reçoit le `total` d'un dev et affiche, avec `@switch` sur un `computed` :
- « Junior » en dessous de 250 points ;
- « Confirmé » en dessous de 350 ;
- « Senior » (en gras) au-delà.

Utilisez `@let` pour stocker le niveau dans le template. Affichez-le dans la carte, après le total.

<details>
<summary>Correction</summary>

`src/app/features/dex/rank-label.ts`

```ts
import { Component, computed, input } from '@angular/core';

type Rank = 'junior' | 'confirme' | 'senior';

/** Niveau d'un dev selon le total de ses statistiques. */
@Component({
  selector: 'app-rank-label',
  template: `
    @let current = rank();
    @switch (current) {
      @case ('junior') {
        <span>Junior</span>
      }
      @case ('confirme') {
        <span>Confirmé</span>
      }
      @default {
        <strong>Senior</strong>
      }
    }
    <span class="muted">({{ total() }} points)</span>
  `,
})
export class RankLabel {
  readonly total = input.required<number>();

  protected readonly rank = computed<Rank>(() => {
    const total = this.total();
    return total < 250 ? 'junior' : total < 350 ? 'confirme' : 'senior';
  });
}
```

Résultat : « Junior (180 points) » pour Stagiairon, « Confirmé (310 points) » pour Kubernaute.
</details>

---

## Check-list

- [ ] Je sais choisir la bonne liaison : `{{ }}`, `[prop]`, `[attr.x]`, `[class.x]`, `[style.x]`, `(event)`.
- [ ] Je sais écrire `@if`, `@for` avec `track`, `@switch`, `@let`.
- [ ] Je sais créer et modifier un signal, et dériver un `computed`.
- [ ] Je sais faire circuler des données avec `input()` et `output()`.
- [ ] Je sais projeter du contenu avec `<ng-content>`.

---

[← 04 · Conception](04-conception.md) · [Sommaire](README.md) · [06 · Services, injection et HTTP →](06-services-http.md)
