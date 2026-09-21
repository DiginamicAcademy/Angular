[← 07 · Directives et pipes](07-directives-pipes.md) · [Sommaire](README.md)

---

# Phase 07 — Routing

## Objectifs

- Déclarer des routes avec chargement différé, redirection et page 404.
- Naviguer avec `routerLink`, `routerLinkActive` et `Router`.
- Lire les paramètres de route et de requête comme des entrées de composant.
- Protéger une route avec une garde et calculer un titre avec un resolver.
- Utiliser `linkedSignal` et `@defer`.

Point d'arrivée : une application à plusieurs pages, avec des URL partageables (`/devs?type=data`, `/devs/11`).

---

## On théorise

### 1. Les routes

```
https://monsite.fr/devs/11?onglet=stats
                 └─┬─┘└┬┘└─────┬─────┘
             chemin  :id   paramètres de requête
```

```ts
export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'devs' }, // redirection
  {
    path: 'devs/:id', // :id est un paramètre
    loadComponent: () =>
      import('./features/detail/dev-detail-page').then((m) => m.DevDetailPage),
    canActivate: [devIdGuard], // garde (exercice 1)
    title: devTitleResolver, // titre calculé (exercice 2)
  },
  {
    path: '**', // tout le reste : page 404
    loadComponent: () =>
      import('./features/not-found/not-found-page').then((m) => m.NotFoundPage),
  },
];
```

- Les routes sont testées **dans l'ordre** : `**` doit être la dernière.
- `loadComponent` avec `import()` : le code de la page est téléchargé à la première visite (*lazy loading*). La build produit un fichier par page.
- `title` : le titre de l'onglet, fixe ou calculé par un *resolver*.

### 2. Afficher et naviguer

| Élément | Rôle |
|---|---|
| `<router-outlet />` | L'emplacement où s'affiche la page de la route active |
| `routerLink="/equipe"` ou `[routerLink]="['/devs', dev.id]"` | Lien interne, sans rechargement |
| `[queryParams]="{ type }"` | Paramètres de requête du lien |
| `routerLinkActive="active"` | Ajoute une classe quand le lien correspond à la route active |
| `ariaCurrentWhenActive="page"` | Ajoute `aria-current="page"` pour les lecteurs d'écran |
| `inject(Router).navigate(['/devs', 12])` | Navigation depuis le code |

### 3. Paramètres = entrées

Avec `provideRouter(routes, withComponentInputBinding())`, les paramètres de route et de requête sont transmis aux **entrées** du composant qui portent le même nom :

```ts
readonly id = input.required({ transform: numberAttribute }); // :id, converti en nombre
readonly type = input<string>();                              // ?type=…
```

Un paramètre d'URL est toujours une chaîne : `numberAttribute` le convertit.

### 4. Gardes et resolvers

| Fonction | Question posée | Réponse |
|---|---|---|
| `CanActivateFn` | Peut-on entrer sur cette route ? | `true`, `false` ou une redirection (`UrlTree`) |
| `CanDeactivateFn` | Peut-on quitter cette page ? | `true` ou `false` (phase 10) |
| `CanMatchFn` | Cette route s'applique-t-elle ? | `true` ou `false` |
| `ResolveFn` | Quelle donnée calculer avant d'afficher ? | Une valeur (ici : le titre) |

Ce sont des **fonctions** ; elles peuvent appeler `inject()`. Une garde améliore l'expérience utilisateur mais ne **sécurise** rien : le code front est modifiable par l'utilisateur.

### 5. `linkedSignal`

Un `linkedSignal` est un signal **modifiable** dont la valeur est **réinitialisée** quand une source change :

```ts
// Par défaut, la meilleure statistique du dev affiché (undefined tant qu'il n'est pas chargé).
// L'utilisateur peut en choisir une autre ; on revient au défaut quand on change de dev.
protected readonly focusedStat = linkedSignal<StatKey | undefined>(() => {
  const dev = this.dev();
  return dev ? bestStat(dev.stats) : undefined;
});
```

`computed` : dérivé, en lecture seule. `signal` : modifiable, sans lien avec une source. `linkedSignal` : les deux.

### 6. `@defer`

```html
@defer (on viewport) {
  <section>… devs du même type …</section>
} @placeholder {
  <p>Devs du même type…</p>
}
```

Le contenu, et le code de ses composants, n'est chargé que lorsque la condition est remplie. Déclencheurs : `on viewport`, `on idle`, `on interaction`, `on hover`, `on timer(2s)`, `when condition`.

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 07.

### Étape 1 — La table des routes

Remplacez `src/app/app.routes.ts` :

`src/app/app.routes.ts`

```ts
import { Routes } from '@angular/router';

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
    title: 'Pokedev · Fiche',
  },
  {
    path: '**',
    loadComponent: () =>
      import('./features/not-found/not-found-page').then((m) => m.NotFoundPage),
    title: 'Pokedev · Page introuvable',
  },
];
```

Dans `src/app/app.config.ts`, activez la liaison des paramètres aux entrées :

`src/app/app.config.ts`

```ts
import { ApplicationConfig, LOCALE_ID, provideBrowserGlobalErrorListeners } from '@angular/core';
import { registerLocaleData } from '@angular/common';
import localeFr from '@angular/common/locales/fr';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { routes } from './app.routes';
import { loadingInterceptor } from './core/http/loading-interceptor';

registerLocaleData(localeFr);

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withInterceptors([loadingInterceptor])),
    { provide: LOCALE_ID, useValue: 'fr' },
  ],
};
```

### Étape 2 — La coquille de l'application

`App` n'affiche plus directement la liste : elle contient l'en-tête, la navigation et le `<router-outlet />`. Remplacez les trois fichiers :

`src/app/app.ts`

```ts
import { Component, inject } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';
import { Loading } from './core/http/loading';
import { Team } from './core/team/team';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {
  protected readonly loading = inject(Loading);
  protected readonly team = inject(Team);
}
```

`src/app/app.html`

```html
@if (loading.active()) {
  <div class="progress" role="progressbar" aria-label="Chargement en cours"></div>
}

<header>
  <a class="brand" routerLink="/devs">Pokedev</a>
  <nav aria-label="Navigation principale">
    <a routerLink="/devs" routerLinkActive="active" ariaCurrentWhenActive="page">Pokédex</a>
    <a routerLink="/equipe" routerLinkActive="active" ariaCurrentWhenActive="page">
      Mon équipe <span class="count">{{ team.size() }}</span>
    </a>
    <a routerLink="/creer" routerLinkActive="active" ariaCurrentWhenActive="page">Créer un dev</a>
  </nav>
</header>

<main>
  <router-outlet />
</main>

<footer>Pokedev · un pokédex de développeurs, tous fictifs.</footer>
```

`src/app/app.css`

```css
header {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  border-bottom: 1px solid var(--border);
}
nav {
  display: flex;
  gap: 1rem;
}
nav a {
  text-decoration: none;
}
nav a.active {
  font-weight: 700;
  text-decoration: underline;
}
.count {
  display: inline-block;
  min-width: 1.4rem;
  border-radius: 999px;
  background: var(--accent);
  color: var(--on-accent);
  text-align: center;
  font-size: 0.8rem;
}
.progress {
  position: fixed;
  inset: 0 0 auto;
  height: 3px;
  background: var(--accent);
  animation: pulse 1s ease-in-out infinite alternate;
}
@keyframes pulse {
  from {
    opacity: 0.3;
  }
  to {
    opacity: 1;
  }
}
@media (prefers-reduced-motion: reduce) {
  .progress {
    animation: none;
  }
}
```

### Étape 3 — Filtrer par l'URL

La liste reçoit le paramètre `?type=` comme entrée et affiche une rangée de filtres. Ses styles passent dans un fichier séparé. Remplacez `src/app/features/dex/dex-page.ts` et créez `dex-page.css` :

`src/app/features/dex/dex-page.ts`

```ts
import { Component, computed, inject, input } from '@angular/core';
import { RouterLink } from '@angular/router';
import { DevRepository } from '../../core/data/dev-repository';
import { isDevType } from '../../core/data/dev-validation';
import { Team } from '../../core/team/team';
import { DEV_TYPES } from '../../domain/dev.model';
import { filterDevs } from '../../domain/dev-rules';
import { DevCard } from './dev-card';
import { TypeColor } from '../../shared/directives/type-color';
import { TypeLabelPipe } from '../../shared/pipes/type-label-pipe';
import { EmptyState } from '../../shared/ui/empty-state';

@Component({
  selector: 'app-dex-page',
  imports: [RouterLink, DevCard, EmptyState, TypeColor, TypeLabelPipe],
  template: `
    <h1>Pokédex</h1>

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

    @if (repository.isLoading()) {
      <p role="status">Chargement du pokédex…</p>
    } @else if (repository.error()) {
      <app-empty-state>
        <p class="error">Impossible de charger le pokédex.</p>
        <button actions type="button" (click)="repository.reload()">Réessayer</button>
      </app-empty-state>
    } @else {
      <p class="muted">{{ devs().length }} dev(s)</p>
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
    }
  `,
  styleUrl: './dex-page.css',
})
export class DexPage {
  protected readonly repository = inject(DevRepository);
  protected readonly team = inject(Team);

  /** Paramètre de requête ?type=…, lié par withComponentInputBinding(). */
  readonly type = input<string>();

  protected readonly types = DEV_TYPES;
  protected readonly selectedType = computed(() => {
    const type = this.type();
    return isDevType(type) ? type : undefined;
  });
  protected readonly devs = computed(() =>
    filterDevs(this.repository.devs(), { query: '', type: this.selectedType() }),
  );
}
```

`src/app/features/dex/dex-page.css`

```css
.filters {
  display: grid;
  gap: 0.25rem;
  max-width: 24rem;
}
.types {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-block: 1rem;
}
.types a {
  padding: 0.2rem 0.75rem;
  border: 2px solid var(--type-color, var(--border));
  border-radius: 999px;
  color: inherit;
  text-decoration: none;
}
.types a.active {
  background: var(--type-color, var(--fg));
  color: var(--bg);
}
.types a[data-type].active {
  color: #fff;
}
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr));
  gap: 1rem;
  padding: 0;
  list-style: none;
}
kbd {
  padding: 0 0.3rem;
  border: 1px solid var(--border);
  border-radius: 0.25rem;
  font-size: 0.8rem;
}
```

- `routerLink="."` : la route courante, avec d'autres paramètres de requête ;
- `isDevType` protège contre une URL modifiée à la main (`?type=cobol`) ;
- le filtre est **dans l'URL** : le lien est partageable et le bouton « Précédent » du navigateur fonctionne.

### Étape 4 — Le lien vers la fiche

Dans la carte, le nom devient un lien. Remplacez `dev-card.ts` et `dev-card.html` :

`src/app/features/dex/dev-card.ts`

```ts
import { Component, computed, input, output } from '@angular/core';
import { RouterLink } from '@angular/router';
import { Dev } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';
import { DexNumberPipe } from '../../shared/pipes/dex-number-pipe';
import { DevAvatar } from '../../shared/ui/dev-avatar';
import { TypeBadge } from '../../shared/ui/type-badge';

@Component({
  selector: 'app-dev-card',
  imports: [RouterLink, DexNumberPipe, DevAvatar, TypeBadge],
  templateUrl: './dev-card.html',
  styleUrl: './dev-card.css',
})
export class DevCard {
  readonly dev = input.required<Dev>();
  readonly inTeam = input(false);
  readonly teamFull = input(false);

  readonly teamToggled = output<number>();

  protected readonly total = computed(() => totalStats(this.dev().stats));
}
```

`src/app/features/dex/dev-card.html`

```html
<article class="card" [class.in-team]="inTeam()">
  <app-dev-avatar [name]="dev().name" [type]="dev().types[0]" />
  <div class="body">
    <p class="number">{{ dev().id | dexNumber }}</p>
    <h3>
      <a [routerLink]="['/devs', dev().id]">{{ dev().name }}</a>
    </h3>
    <p class="title">{{ dev().title }}</p>
    <p class="types">
      @for (type of dev().types; track type) {
        <app-type-badge [type]="type" />
      }
    </p>
    <p class="total">Total : {{ total() }}</p>
  </div>
  <button
    type="button"
    [disabled]="!inTeam() && teamFull()"
    [attr.aria-pressed]="inTeam()"
    (click)="teamToggled.emit(dev().id)"
  >
    {{ inTeam() ? 'Retirer de l’équipe' : 'Ajouter à l’équipe' }}
  </button>
</article>
```

### Étape 5 — La fiche d'un dev

Créez le dossier `src/app/features/detail/` et ses trois fichiers :

`src/app/features/detail/dev-detail-page.ts`

```ts
import { Component, computed, inject, input, linkedSignal, numberAttribute } from '@angular/core';
import { RouterLink } from '@angular/router';
import { DevRepository } from '../../core/data/dev-repository';
import { Team } from '../../core/team/team';
import { STAT_KEYS, StatKey } from '../../domain/dev.model';
import { bestStat, previousEvolution, totalStats } from '../../domain/dev-rules';
import { STAT_LABELS } from '../../domain/labels';
import { DexNumberPipe } from '../../shared/pipes/dex-number-pipe';
import { TypeColor } from '../../shared/directives/type-color';
import { DevAvatar } from '../../shared/ui/dev-avatar';
import { EmptyState } from '../../shared/ui/empty-state';
import { StatBar } from '../../shared/ui/stat-bar';
import { TypeBadge } from '../../shared/ui/type-badge';

@Component({
  selector: 'app-dev-detail-page',
  imports: [RouterLink, DexNumberPipe, TypeColor, DevAvatar, EmptyState, StatBar, TypeBadge],
  templateUrl: './dev-detail-page.html',
  styleUrl: './dev-detail-page.css',
})
export class DevDetailPage {
  private readonly repository = inject(DevRepository);
  protected readonly team = inject(Team);

  /** Paramètre de route :id, converti en nombre. */
  readonly id = input.required({ transform: numberAttribute });

  protected readonly statKeys = STAT_KEYS;
  protected readonly statLabels = STAT_LABELS;
  protected readonly isLoading = this.repository.isLoading;

  protected readonly dev = computed(() => this.repository.byId(this.id()));
  protected readonly total = computed(() => {
    const dev = this.dev();
    return dev ? totalStats(dev.stats) : 0;
  });
  protected readonly previous = computed(() => {
    const dev = this.dev();
    return dev ? previousEvolution(this.repository.devs(), dev) : undefined;
  });
  protected readonly next = computed(() => {
    const target = this.dev()?.evolvesTo;
    return target === undefined ? undefined : this.repository.byId(target);
  });

  /**
   * Statistique mise en avant : par défaut la meilleure du dev affiché.
   * L'utilisateur peut en choisir une autre ; le choix est réinitialisé
   * quand on passe à un autre dev.
   */
  protected readonly focusedStat = linkedSignal<StatKey | undefined>(() => {
    const dev = this.dev();
    return dev ? bestStat(dev.stats) : undefined;
  });

  protected readonly devCount = computed(() => this.repository.devs().length);

  /** Rang du dev pour la statistique mise en avant (1 = meilleur). */
  protected readonly focusedRank = computed(() => {
    const dev = this.dev();
    const key = this.focusedStat();
    if (!dev || !key) {
      return undefined;
    }
    return this.repository.devs().filter((other) => other.stats[key] > dev.stats[key]).length + 1;
  });

  /** Autres devs partageant le type principal. */
  protected readonly sameType = computed(() => {
    const dev = this.dev();
    if (!dev) {
      return [];
    }
    return this.repository
      .devs()
      .filter((other) => other.id !== dev.id && other.types.includes(dev.types[0]));
  });
}
```

`src/app/features/detail/dev-detail-page.html`

```html
@let current = dev();

@if (current) {
  <article [appTypeColor]="current.types[0]">
    <header class="identity">
      <app-dev-avatar [name]="current.name" [type]="current.types[0]" style="--avatar-size: 6rem" />
      <div>
        <p class="muted">{{ current.id | dexNumber }}</p>
        <h1>{{ current.name }}</h1>
        <p>{{ current.title }}</p>
        <p class="types">
          @for (type of current.types; track type) {
            <app-type-badge [type]="type" />
          }
        </p>
      </div>
    </header>

    <blockquote>« {{ current.catchphrase }} »</blockquote>

    <section aria-labelledby="stats-title">
      <h2 id="stats-title">Statistiques · total {{ total() }}</h2>
      <div class="stats">
        @for (key of statKeys; track key) {
          <app-stat-bar
            [label]="statLabels[key]"
            [value]="current.stats[key]"
            [highlight]="focusedStat() === key"
          />
        }
      </div>

      <div class="stat-picker" role="group" aria-label="Statistique mise en avant">
        @for (key of statKeys; track key) {
          <button type="button" [attr.aria-pressed]="focusedStat() === key" (click)="focusedStat.set(key)">
            {{ statLabels[key] }}
          </button>
        }
      </div>
      @if (focusedStat(); as key) {
        <p>
          {{ statLabels[key] }} : {{ current.stats[key] }}/100, rang {{ focusedRank() }} sur
          {{ devCount() }}.
        </p>
      }
    </section>

    <section aria-labelledby="languages-title">
      <h2 id="languages-title">Langages</h2>
      <ul class="languages">
        @for (language of current.languages; track language) {
          <li>{{ language }}</li>
        } @empty {
          <li class="muted">Aucun langage renseigné.</li>
        }
      </ul>
    </section>

    @if (previous() || next()) {
      <section aria-labelledby="evolution-title">
        <h2 id="evolution-title">Évolution</h2>
        <p class="evolution">
          @if (previous(); as before) {
            <a [routerLink]="['/devs', before.id]">← {{ before.name }}</a>
          }
          <strong>{{ current.name }}</strong>
          @if (next(); as after) {
            <a [routerLink]="['/devs', after.id]">{{ after.name }} →</a>
          }
        </p>
      </section>
    }

    <p>
      <button
        type="button"
        [disabled]="!team.has(current.id) && team.isFull()"
        (click)="team.toggle(current.id)"
      >
        {{ team.has(current.id) ? 'Retirer de l’équipe' : 'Ajouter à l’équipe' }}
      </button>
    </p>

    @defer (on viewport) {
      <section aria-labelledby="same-type-title">
        <h2 id="same-type-title">Du même type</h2>
        <ul class="same-type">
          @for (other of sameType(); track other.id) {
            <li>
              <a [routerLink]="['/devs', other.id]">{{ other.id | dexNumber }} {{ other.name }}</a>
            </li>
          } @empty {
            <li class="muted">Aucun autre dev de ce type.</li>
          }
        </ul>
      </section>
    } @placeholder {
      <p class="muted">Devs du même type…</p>
    }
  </article>
} @else if (isLoading()) {
  <p role="status">Chargement…</p>
} @else {
  <app-empty-state>
    <p>Aucun dev ne porte le numéro {{ id() | dexNumber }}.</p>
    <a actions routerLink="/devs">Retour au pokédex</a>
  </app-empty-state>
}
```

`src/app/features/detail/dev-detail-page.css`

```css
.identity {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}
.identity p,
.identity h1 {
  margin: 0;
}
.types {
  display: flex;
  gap: 0.25rem;
  margin-top: 0.5rem !important;
}
blockquote {
  margin: 1.5rem 0;
  padding-left: 1rem;
  border-left: 4px solid var(--type-color);
  font-style: italic;
}
.stats {
  display: grid;
  gap: 0.4rem;
  max-width: 32rem;
}
.stat-picker {
  display: flex;
  flex-wrap: wrap;
  gap: 0.25rem;
  margin-top: 1rem;
}
.stat-picker [aria-pressed='true'] {
  background: var(--type-color);
  color: #fff;
}
.languages,
.same-type {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  padding: 0;
  list-style: none;
}
.languages li {
  padding: 0.1rem 0.5rem;
  border: 1px solid var(--border);
  border-radius: 0.25rem;
}
.evolution {
  display: flex;
  gap: 1rem;
  align-items: center;
}
```

- `@let current = dev();` évite de rappeler le signal et permet le narrowing dans `@if` ;
- `focusedStat` est un `linkedSignal` : il suit la meilleure statistique du dev affiché, mais l'utilisateur peut en choisir une autre ;
- `@defer (on viewport)` : la section « Du même type » n'est rendue que lorsqu'elle devient visible ;
- trois états : dev trouvé, chargement, numéro inconnu.

### Étape 6 — La page 404

Créez `src/app/features/not-found/not-found-page.ts` :

`src/app/features/not-found/not-found-page.ts`

```ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';
import { EmptyState } from '../../shared/ui/empty-state';

@Component({
  selector: 'app-not-found-page',
  imports: [RouterLink, EmptyState],
  template: `
    <h1>Page introuvable</h1>
    <app-empty-state>
      <p>Ce dev s'est échappé dans la nature.</p>
      <a actions routerLink="/devs">Retour au pokédex</a>
    </app-empty-state>
  `,
})
export class NotFoundPage {}
```

### Résultat attendu

Faites le parcours suivant :
1. http://localhost:4200 redirige vers `/devs` ; l'onglet du navigateur affiche « Pokedev · Pokédex ».
2. Cliquez sur le filtre « Data » : l'URL devient `/devs?type=data` et 3 devs restent. Le bouton « Précédent » du navigateur revient aux 21 devs.
3. Saisissez `/devs?type=cobol` dans la barre d'adresse : le filtre est ignoré, les 21 devs s'affichent.
4. Cliquez sur « Requêtor » : l'URL devient `/devs/11`. La fiche affiche « Statistiques · total 335 » et « Code : 65/100, rang 4 sur 21. ».
5. Cliquez sur « Café » : « Café : 55/100 ». Cliquez ensuite sur « ← Tabulis » : la statistique mise en avant revient à la meilleure de Tabulis, « Communication : 55/100 ». C'est l'effet du `linkedSignal`.
6. Faites défiler jusqu'en bas : la section « Du même type » apparaît.
7. Ouvrez `/devs/999` : « Aucun dev ne porte le numéro #999. » ; ouvrez `/nimporte-quoi` : « Page introuvable ».

Les liens « Mon équipe » et « Créer un dev » mènent pour l'instant à la page 404 : leurs routes arrivent aux phases 09 et 10.

---

## Vous faites

**Exercice 1 — Une garde sur le numéro.** Générez une garde avec `npx ng g guard core/navigation/dev-id --skip-tests` (type `CanActivate`, proposé par défaut). `/devs/abc`, `/devs/0` ou `/devs/2.5` doivent rediriger vers `/introuvable` ; un entier positif est accepté. Déclarez-la sur la route `devs/:id`.

<details>
<summary>Correction</summary>

`src/app/core/navigation/dev-id-guard.ts`

```ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

/** Refuse les numéros invalides (/devs/abc, /devs/0) et redirige vers la page 404. */
export const devIdGuard: CanActivateFn = (route) => {
  const id = Number(route.paramMap.get('id'));
  return Number.isInteger(id) && id > 0 ? true : inject(Router).parseUrl('/introuvable');
};
```

`/introuvable` n'est pas une route déclarée : c'est la route `**` qui l'affiche. La table des routes, avec aussi le resolver de l'exercice 2 :

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
    path: '**',
    loadComponent: () =>
      import('./features/not-found/not-found-page').then((m) => m.NotFoundPage),
    title: 'Pokedev · Page introuvable',
  },
];
```

Résultat : `/devs/abc` affiche « Page introuvable » et l'URL devient `/introuvable`.
</details>

**Exercice 2 — Un titre calculé.** Générez un resolver avec `npx ng g resolver core/navigation/dev-title --skip-tests`. Il renvoie « Pokedev · #011 » pour `/devs/11`. Utilisez-le comme `title` de la route.

<details>
<summary>Correction</summary>

`src/app/core/navigation/dev-title-resolver.ts`

```ts
import { ResolveFn } from '@angular/router';
import { formatDexNumber } from '../../domain/dev-rules';

/** Titre de l'onglet pour la page d'un dev : « Pokedev · #007 ». */
export const devTitleResolver: ResolveFn<string> = (route) =>
  `Pokedev · ${formatDexNumber(Number(route.paramMap.get('id')))}`;
```

Résultat : sur `/devs/11`, l'onglet du navigateur et l'historique affichent « Pokedev · #011 ».
</details>

**Exercice 3 — Observer `@defer`.** Sur la fiche d'un dev, réduisez la hauteur de la fenêtre pour que la section « Du même type » soit hors écran : le texte « Devs du même type… » est affiché à sa place. Faites défiler. Remplacez ensuite `on viewport` par `on interaction` et comparez.

<details>
<summary>Correction</summary>

Avec `on interaction`, le contenu n'apparaît qu'après un clic ou une touche **sur le texte de remplacement**. `@placeholder` est donc indispensable : sans lui, il n'y aurait rien sur quoi interagir. Remettez `on viewport` à la fin de l'exercice.
</details>

---

## Check-list

- [ ] Je sais déclarer des routes avec chargement différé, redirection et `**`.
- [ ] Je sais lire un paramètre de route et un paramètre de requête avec `input()`.
- [ ] Je sais écrire une garde qui redirige et un resolver.
- [ ] Je sais quand utiliser `linkedSignal` plutôt que `computed` ou `signal`.
- [ ] Je sais différer un bloc avec `@defer`.

---

[← 07 · Directives et pipes](07-directives-pipes.md) · [Sommaire](README.md) · [09 · État, persistance et architecture →](09-etat-persistance-architecture.md)
