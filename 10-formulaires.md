[← 09 · État, persistance et architecture](09-etat-persistance-architecture.md) · [Sommaire](README.md)

---

# Phase 09 — Formulaires avec Signal Forms

## Objectifs

- Créer un formulaire à partir d'un signal avec `form()` et `[formField]`.
- Déclarer des règles de validation, dont une validation croisée entre champs.
- Afficher les erreurs au bon moment avec l'état des champs.
- Retarder la mise à jour d'un champ avec `debounce()`.
- Accéder à un élément du template avec `viewChild` et écouter le clavier sur le document.
- Empêcher de quitter une page avec des saisies non enregistrées.

Point d'arrivée : une recherche instantanée dans le pokédex et un formulaire de création de dev.

---

## On théorise

### 1. Le principe des Signal Forms

Le **modèle** du formulaire est un signal. `form()` construit à partir de lui un **arbre de champs**, en appliquant un **schéma** de règles :

```ts
protected readonly draft = signal({ name: '', title: '' });          // le modèle

protected readonly devForm = form(this.draft, (path) => {            // le schéma
  required(path.name, { message: 'Le nom est obligatoire.' });
  maxLength(path.name, 30, { message: 'Au plus 30 caractères.' });
});
```

```html
<input id="name" [formField]="devForm.name" />
```

- La saisie modifie le signal `draft`, et une modification de `draft` met à jour le champ : la liaison est **dans les deux sens**.
- `devForm.name` est un champ ; `devForm.name()` renvoie son **état** :

| État | Signification |
|---|---|
| `value()` | la valeur |
| `errors()` | la liste des erreurs (`kind`, `message`) |
| `valid()`, `invalid()` | le champ, et ses sous-champs, respectent les règles ou non |
| `touched()` | l'utilisateur a quitté le champ au moins une fois |
| `dirty()` | l'utilisateur a modifié la valeur |
| `submitting()` | une soumission est en cours |

- `devForm()` renvoie l'état du formulaire entier.

### 2. Les règles

| Règle | Exemple |
|---|---|
| `required` | `required(path.title)` |
| `minLength`, `maxLength` | `minLength(path.name, 2)` |
| `min`, `max` | `max(path.stats.code, 100)` |
| `pattern`, `email` | `email(path.contact)` |
| `validate` | une règle personnalisée, qui renvoie une erreur ou `undefined` |
| `debounce` | `debounce(path.query, 200)` : le modèle n'est mis à jour qu'après 200 ms sans frappe |

Une règle personnalisée reçoit un contexte : `value()` est la valeur du champ, `valueOf(autreChemin)` celle d'un autre champ. C'est ainsi qu'on écrit une **validation croisée** :

```ts
validate(path.secondaryType, ({ value, valueOf }) =>
  value() !== '' && value() === valueOf(path.primaryType)
    ? { kind: 'duplicate', message: 'Le type secondaire doit différer du type principal.' }
    : undefined,
);
```

Une règle peut porter sur un **groupe** (`path.stats`) : elle voit alors l'objet entier.

Les attributs HTML `min`, `max`, `required`… sont **interdits** sur un élément qui porte `[formField]` (erreur de compilation NG8022) : ils sont déduits du schéma.

### 3. La soumission

```ts
protected async save(event: Event): Promise<void> {
  event.preventDefault();                           // pas de rechargement de page
  await submit(this.devForm, async (field) => {     // n'appelle l'action que si le formulaire est valide
    this.repository.add(draftToDev(field().value()));
    return undefined;                               // ou des erreurs renvoyées par un serveur
  });
}
```

`submit` marque tous les champs comme touchés, puis n'exécute l'action que si le formulaire est valide. Une deuxième soumission lancée pendant la première est ignorée.

### 4. Références vers le template

```ts
private readonly searchInput = viewChild.required<ElementRef<HTMLInputElement>>('searchInput');
// template : <input #searchInput … />
this.searchInput().nativeElement.focus();
```

`viewChild` est un signal. On l'utilise pour les actions impératives sur le DOM : focus, défilement, mesure.

Un écouteur sur le document se déclare dans `host` :

```ts
host: { '(document:keydown./)': 'focusSearch($event)' }
```

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 09.

### Étape 1 — La recherche dans le pokédex

La liste reçoit un champ de recherche filtré en direct, et le raccourci clavier `/` pour y placer le curseur. Son template passe dans un fichier séparé. Remplacez `src/app/features/dex/dex-page.ts` et créez `dex-page.html` :

`src/app/features/dex/dex-page.ts`

```ts
import { Component, ElementRef, computed, inject, input, signal, viewChild } from '@angular/core';
import { RouterLink } from '@angular/router';
import { FormField, debounce, form } from '@angular/forms/signals';
import { DevRepository } from '../../core/data/dev-repository';
import { Team } from '../../core/team/team';
import { DEV_TYPES } from '../../domain/dev.model';
import { filterDevs } from '../../domain/dev-rules';
import { isDevType } from '../../core/data/dev-validation';
import { TypeLabelPipe } from '../../shared/pipes/type-label-pipe';
import { TypeColor } from '../../shared/directives/type-color';
import { EmptyState } from '../../shared/ui/empty-state';
import { DevCard } from './dev-card';

@Component({
  selector: 'app-dex-page',
  imports: [FormField, RouterLink, DevCard, EmptyState, TypeColor, TypeLabelPipe],
  templateUrl: './dex-page.html',
  styleUrl: './dex-page.css',
  host: { '(document:keydown./)': 'focusSearch($event)' },
})
export class DexPage {
  private readonly repository = inject(DevRepository);
  protected readonly team = inject(Team);

  /** Paramètre de requête ?type=… lié automatiquement par le routeur. */
  readonly type = input<string>();

  protected readonly types = DEV_TYPES;
  protected readonly isLoading = this.repository.isLoading;
  protected readonly error = this.repository.error;

  protected readonly search = signal({ query: '' });
  protected readonly searchForm = form(this.search, (path) => {
    debounce(path.query, 200);
  });

  private readonly searchInput = viewChild.required<ElementRef<HTMLInputElement>>('searchInput');

  protected readonly selectedType = computed(() => {
    const type = this.type();
    return isDevType(type) ? type : undefined;
  });

  protected readonly devs = computed(() =>
    filterDevs(this.repository.devs(), {
      query: this.search().query,
      type: this.selectedType(),
    }),
  );

  protected reload(): void {
    this.repository.reload();
  }

  protected focusSearch(event: Event): void {
    const target = event.target as HTMLElement | null;
    if (target?.closest('input, textarea, select')) {
      return;
    }
    event.preventDefault();
    this.searchInput().nativeElement.focus();
  }

  protected resetSearch(): void {
    this.search.set({ query: '' });
  }
}
```

`src/app/features/dex/dex-page.html`

```html
<h1>Pokédex</h1>

<form class="filters" (submit)="$event.preventDefault()">
  <label for="search">Rechercher un dev <kbd>/</kbd></label>
  <input
    #searchInput
    id="search"
    type="search"
    autocomplete="off"
    placeholder="Nom, poste ou langage"
    [formField]="searchForm.query"
  />
</form>

<nav class="types" aria-label="Filtrer par type">
  <a routerLink="." [queryParams]="{}" [class.active]="!selectedType()">Tous</a>
  @for (type of types; track type) {
    <a
      routerLink="."
      [queryParams]="{ type }"
      [appTypeColor]="type"
      [class.active]="selectedType() === type"
      [attr.aria-current]="selectedType() === type ? 'true' : null"
    >
      {{ type | typeLabel }}
    </a>
  }
</nav>

@if (isLoading()) {
  <p role="status">Chargement du pokédex…</p>
} @else if (error()) {
  <app-empty-state>
    <p class="error">Impossible de charger le pokédex.</p>
    <button actions type="button" (click)="reload()">Réessayer</button>
  </app-empty-state>
} @else {
  <p class="muted" aria-live="polite">{{ devs().length }} dev(s)</p>

  @if (devs().length > 0) {
    <ul class="grid">
      @for (dev of devs(); track dev.id) {
        <li>
          <app-dev-card
            [dev]="dev"
            [inTeam]="team.has(dev.id)"
            [teamFull]="team.isFull()"
            (teamToggled)="team.toggle($event)"
          />
        </li>
      }
    </ul>
  } @else {
    <app-empty-state>
      <p>Aucun dev ne correspond à la recherche.</p>
      <button actions type="button" (click)="resetSearch()">Effacer la recherche</button>
    </app-empty-state>
  }
}
```

- `search` est le modèle, `searchForm` le formulaire ; `devs` lit `search()`, pas le champ ;
- `debounce(path.query, 200)` : la liste ne se recalcule qu'après 200 ms sans frappe ;
- `focusSearch` ignore la touche `/` quand on écrit déjà dans un champ ;
- `(submit)="$event.preventDefault()"` : la touche Entrée ne recharge pas la page.

Résultat attendu :
1. Appuyez sur `/` : le curseur est dans le champ de recherche.
2. Tapez « kube » : « 1 dev(s) », Kubernaute.
3. Remplacez par « python » : 4 devs (Tabulis, Requêtor, Chiffrinou, Datalga). Cliquez sur le filtre « Data » : 3 devs.
4. Tapez « zzz » : « Aucun dev ne correspond à la recherche. » et un bouton « Effacer la recherche ».

### Étape 2 — Le brouillon d'un dev

Le formulaire de création manipule un **brouillon**, dont la forme diffère de celle d'un `Dev` : les langages sont saisis dans un seul champ, le type secondaire peut être vide. Des fonctions pures convertissent l'un en l'autre. Créez `src/app/features/create/dev-draft.ts` :

`src/app/features/create/dev-draft.ts`

```ts
import { Dev, DevStats, DevType } from '../../domain/dev.model';

/** Valeurs saisies dans le formulaire de création. */
export interface DevDraft {
  name: string;
  title: string;
  primaryType: DevType;
  secondaryType: DevType | '';
  stats: DevStats;
  /** Langages séparés par des virgules. */
  languages: string;
  catchphrase: string;
}

export function emptyDraft(): DevDraft {
  return {
    name: '',
    title: '',
    primaryType: 'frontend',
    secondaryType: '',
    stats: { code: 50, debug: 50, archi: 50, tests: 50, communication: 50, cafe: 50 },
    languages: '',
    catchphrase: '',
  };
}

/** « TypeScript, , SQL » → ['TypeScript', 'SQL'] */
export function splitLanguages(text: string): string[] {
  return text
    .split(',')
    .map((language) => language.trim())
    .filter((language) => language !== '');
}

export function draftToDev(draft: DevDraft): Omit<Dev, 'id' | 'custom'> {
  return {
    name: draft.name.trim(),
    title: draft.title.trim(),
    types: draft.secondaryType ? [draft.primaryType, draft.secondaryType] : [draft.primaryType],
    stats: { ...draft.stats },
    languages: splitLanguages(draft.languages),
    catchphrase: draft.catchphrase.trim(),
  };
}
```

---

## Vous faites

**Exercice 1 — Le formulaire de création.** Créez `src/app/features/create/create-dev-page.ts` (avec `.html` et `.css`). Règles :
- nom : obligatoire, 2 à 30 caractères, **unique** dans le pokédex (sans tenir compte de la casse) ;
- poste : obligatoire, 50 caractères au plus ;
- type principal : une liste ; type secondaire : « Aucun » ou un type **différent** du principal ;
- six statistiques entre 0 et 100, dont le **total** ne dépasse pas 420 ; le total s'affiche en direct ;
- langages : au moins un, séparés par des virgules ;
- phrase fétiche : 120 caractères au plus.

Les erreurs du nom, du poste et des langages s'affichent quand le champ a été touché. Le bouton est désactivé tant que le formulaire est invalide. Après l'enregistrement, l'application ouvre la fiche du nouveau dev.

La route `creer` est ajoutée à l'exercice 2.

<details>
<summary>Correction</summary>

`src/app/features/create/create-dev-page.ts`

```ts
import { Component, computed, inject, signal } from '@angular/core';
import { Router } from '@angular/router';
import {
  FormField,
  form,
  max,
  maxLength,
  min,
  minLength,
  required,
  submit,
  validate,
} from '@angular/forms/signals';
import { DevRepository } from '../../core/data/dev-repository';
import { HasUnsavedChanges } from '../../core/navigation/unsaved-changes-guard';
import { DEV_TYPES, MAX_STAT, MAX_TOTAL, STAT_KEYS } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';
import { STAT_LABELS } from '../../domain/labels';
import { TypeLabelPipe } from '../../shared/pipes/type-label-pipe';
import { DevDraft, draftToDev, emptyDraft, splitLanguages } from './dev-draft';

@Component({
  selector: 'app-create-dev-page',
  imports: [FormField, TypeLabelPipe],
  templateUrl: './create-dev-page.html',
  styleUrl: './create-dev-page.css',
})
export class CreateDevPage implements HasUnsavedChanges {
  private readonly repository = inject(DevRepository);
  private readonly router = inject(Router);

  protected readonly types = DEV_TYPES;
  protected readonly statKeys = STAT_KEYS;
  protected readonly statLabels = STAT_LABELS;
  protected readonly maxTotal = MAX_TOTAL;

  protected readonly draft = signal<DevDraft>(emptyDraft());
  private readonly saved = signal(false);

  protected readonly devForm = form(this.draft, (path) => {
    required(path.name, { message: 'Le nom est obligatoire.' });
    minLength(path.name, 2, { message: 'Au moins 2 caractères.' });
    maxLength(path.name, 30, { message: 'Au plus 30 caractères.' });
    validate(path.name, ({ value }) => {
      const name = value().trim().toLowerCase();
      return this.repository.devs().some((dev) => dev.name.toLowerCase() === name)
        ? { kind: 'unique', message: 'Ce nom est déjà pris.' }
        : undefined;
    });

    required(path.title, { message: 'Le poste est obligatoire.' });
    maxLength(path.title, 50, { message: 'Au plus 50 caractères.' });

    validate(path.secondaryType, ({ value, valueOf }) =>
      value() !== '' && value() === valueOf(path.primaryType)
        ? { kind: 'duplicate', message: 'Le type secondaire doit différer du type principal.' }
        : undefined,
    );

    for (const key of STAT_KEYS) {
      min(path.stats[key], 0, { message: 'Minimum : 0.' });
      max(path.stats[key], MAX_STAT, { message: `Maximum : ${MAX_STAT}.` });
    }
    validate(path.stats, ({ value }) =>
      totalStats(value()) > MAX_TOTAL
        ? { kind: 'total', message: `Le total ne doit pas dépasser ${MAX_TOTAL}.` }
        : undefined,
    );

    validate(path.languages, ({ value }) =>
      splitLanguages(value()).length === 0
        ? { kind: 'languages', message: 'Indiquez au moins un langage.' }
        : undefined,
    );
    maxLength(path.catchphrase, 120, { message: 'Au plus 120 caractères.' });
  });

  protected readonly total = computed(() => totalStats(this.draft().stats));

  hasUnsavedChanges(): boolean {
    return this.devForm().dirty() && !this.saved();
  }

  protected async save(event: Event): Promise<void> {
    event.preventDefault();
    await submit(this.devForm, async (field) => {
      const id = this.repository.add(draftToDev(field().value()));
      this.saved.set(true);
      await this.router.navigate(['/devs', id]);
      return undefined;
    });
  }
}
```

`src/app/features/create/create-dev-page.html`

```html
<h1>Créer un dev</h1>

<form (submit)="save($event)" novalidate>
  <div class="field">
    <label for="name">Nom</label>
    <input id="name" type="text" [formField]="devForm.name" />
    @if (devForm.name().touched()) {
      @for (error of devForm.name().errors(); track error.kind) {
        <p class="error">{{ error.message }}</p>
      }
    }
  </div>

  <div class="field">
    <label for="title">Poste</label>
    <input id="title" type="text" [formField]="devForm.title" />
    @if (devForm.title().touched()) {
      @for (error of devForm.title().errors(); track error.kind) {
        <p class="error">{{ error.message }}</p>
      }
    }
  </div>

  <div class="row">
    <div class="field">
      <label for="primaryType">Type principal</label>
      <select id="primaryType" [formField]="devForm.primaryType">
        @for (type of types; track type) {
          <option [value]="type">{{ type | typeLabel }}</option>
        }
      </select>
    </div>
    <div class="field">
      <label for="secondaryType">Type secondaire</label>
      <select id="secondaryType" [formField]="devForm.secondaryType">
        <option value="">Aucun</option>
        @for (type of types; track type) {
          <option [value]="type">{{ type | typeLabel }}</option>
        }
      </select>
      @for (error of devForm.secondaryType().errors(); track error.kind) {
        <p class="error">{{ error.message }}</p>
      }
    </div>
  </div>

  <fieldset>
    <legend>Statistiques · total {{ total() }}/{{ maxTotal }}</legend>
    <div class="stats">
      @for (key of statKeys; track key) {
        <div class="field">
          <label [for]="'stat-' + key">{{ statLabels[key] }}</label>
          <input [id]="'stat-' + key" type="number" [formField]="devForm.stats[key]" />
          @for (error of devForm.stats[key]().errors(); track error.kind) {
            <p class="error">{{ error.message }}</p>
          }
        </div>
      }
    </div>
    @for (error of devForm.stats().errors(); track error.kind) {
      <p class="error">{{ error.message }}</p>
    }
  </fieldset>

  <div class="field">
    <label for="languages">Langages (séparés par des virgules)</label>
    <input id="languages" type="text" [formField]="devForm.languages" />
    @if (devForm.languages().touched()) {
      @for (error of devForm.languages().errors(); track error.kind) {
        <p class="error">{{ error.message }}</p>
      }
    }
  </div>

  <div class="field">
    <label for="catchphrase">Phrase fétiche</label>
    <textarea id="catchphrase" rows="2" [formField]="devForm.catchphrase"></textarea>
    @for (error of devForm.catchphrase().errors(); track error.kind) {
      <p class="error">{{ error.message }}</p>
    }
  </div>

  <button type="submit" [disabled]="devForm().invalid() || devForm().submitting()">
    Ajouter au pokédex
  </button>
</form>
```

`src/app/features/create/create-dev-page.css`

```css
form {
  display: grid;
  gap: 1rem;
  max-width: 36rem;
}
.field {
  display: grid;
  gap: 0.25rem;
}
.field p {
  margin: 0;
}
.row,
.stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(10rem, 1fr));
  gap: 1rem;
}
fieldset {
  border: 1px solid var(--border);
  border-radius: 0.5rem;
}
button[type='submit'] {
  justify-self: start;
}
```

Remarques :
- la règle d'unicité lit `this.repository.devs()` : elle est réévaluée si la liste change ;
- les règles des statistiques sont déclarées dans une boucle sur `STAT_KEYS` ;
- l'erreur du total est portée par le groupe `devForm.stats` ;
- les champs numériques n'ont pas d'attributs `min` ou `max` : ils sont interdits avec `[formField]` (erreur NG8022) et déduits du schéma.
</details>

**Exercice 2 — Ne pas perdre la saisie.** Générez une garde avec `npx ng g guard core/navigation/unsaved-changes --implements CanDeactivate --skip-tests`. Elle demande confirmation (`confirm`) si le composant signale des modifications non enregistrées. `CreateDevPage` implémente l'interface `HasUnsavedChanges` : il y a des modifications si le formulaire est `dirty` et n'a pas été enregistré. Ajoutez la route `creer` (titre « Pokedev · Créer un dev ») protégée par cette garde.

<details>
<summary>Correction</summary>

`src/app/core/navigation/unsaved-changes-guard.ts`

```ts
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

/** Demande confirmation avant de quitter une page qui contient des saisies non enregistrées. */
export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) =>
  !component.hasUnsavedChanges() ||
  confirm('Vos saisies ne sont pas enregistrées. Quitter quand même ?');
```

La table des routes complète :

`src/app/app.routes.ts`

```ts
import { Routes } from '@angular/router';
import { devIdGuard } from './core/navigation/dev-id-guard';
import { devTitleResolver } from './core/navigation/dev-title-resolver';
import { unsavedChangesGuard } from './core/navigation/unsaved-changes-guard';

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
    path: 'creer',
    loadComponent: () =>
      import('./features/create/create-dev-page').then((m) => m.CreateDevPage),
    canDeactivate: [unsavedChangesGuard],
    title: 'Pokedev · Créer un dev',
  },
  {
    path: '**',
    loadComponent: () =>
      import('./features/not-found/not-found-page').then((m) => m.NotFoundPage),
    title: 'Pokedev · Page introuvable',
  },
];
```
</details>

**Exercice 3 — Tester à la main.** Sur http://localhost:4200/creer :
1. saisissez « juniorax » comme nom, puis cliquez ailleurs ;
2. choisissez « Data » comme type principal et comme type secondaire ;
3. mettez 90 dans les statistiques Code, Debug, Architecture, Tests et Communication ;
4. cliquez sur « Pokédex » dans l'en-tête ;
5. remettez 50 dans ces cinq statistiques, saisissez le nom « Testeuse », le poste « Ingénieure qualité », les langages « Java, Gherkin », le type secondaire « Back-end », puis enregistrez ;
6. rechargez la fiche, puis cherchez « testeuse » dans le pokédex.

<details>
<summary>Résultats attendus</summary>

1. « Ce nom est déjà pris. »
2. « Le type secondaire doit différer du type principal. »
3. « Statistiques · total 500/420 » et « Le total ne doit pas dépasser 420. »
4. La boîte de confirmation « Vos saisies ne sont pas enregistrées. Quitter quand même ? » ; « Annuler » garde la page.
5. La fiche de Testeuse s'ouvre, avec le numéro #022.
6. Testeuse est toujours là après rechargement (clé `pokedev.custom-devs.v1`) et la recherche la trouve.
</details>

---

## Check-list

- [ ] Je sais créer un formulaire avec `form()` et `[formField]`.
- [ ] Je sais écrire des règles, dont une validation croisée et une règle de groupe.
- [ ] Je sais afficher les erreurs avec `touched()` et `errors()`.
- [ ] Je sais utiliser `debounce()` et `submit()`.
- [ ] Je sais utiliser `viewChild` et un écouteur `host`.
- [ ] Je sais écrire une garde `CanDeactivateFn`.

---

[← 09 · État, persistance et architecture](09-etat-persistance-architecture.md) · [Sommaire](README.md) · [11 · Tests →](11-tests.md)
