[← 06 · Services, injection et HTTP](06-services-http.md) · [Sommaire](README.md)

---

# Phase 06 — Directives et pipes

## Objectifs

- Écrire une directive d'attribut avec des liaisons sur l'élément hôte.
- Réutiliser une directive dans un composant avec `hostDirectives`.
- Écrire un pipe personnalisé et utiliser les pipes intégrés.
- Configurer la locale française.

Point d'arrivée : des cartes colorées selon le type, avec numéros au format `#007`, libellés français et avatars.

---

## On théorise

### 1. Composant, directive, pipe

| Élément | Rôle | Exemple |
|---|---|---|
| Composant | Un morceau d'interface avec son template | `<app-dev-card>` |
| Directive d'attribut | Ajoute un comportement ou une apparence à un élément existant | `<span [appTypeColor]="'data'">` |
| Pipe | Transforme une valeur pour l'affichage | `{{ 7 \| dexNumber }}` → `#007` |

Un composant est une directive qui possède un template.

Les **directives structurelles** historiques (`*ngIf`, `*ngFor`) sont remplacées par le control flow (`@if`, `@for`). On écrit aujourd'hui surtout des directives d'attribut.

### 2. Une directive d'attribut

```ts
@Directive({
  selector: '[appTypeColor]',               // s'applique à tout élément portant cet attribut
  host: {
    '[style.--type-color]': 'color()',      // liaison sur l'élément hôte
    '[attr.data-type]': 'appTypeColor()',
  },
})
export class TypeColor {
  readonly appTypeColor = input.required<DevType>(); // même nom que le sélecteur
  protected readonly color = computed(() => TYPE_COLORS[this.appTypeColor()]);
}
```

- La propriété `host` déclare des liaisons (`[…]`) et des écouteurs (`(…)`) sur l'élément qui porte la directive.
- Une entrée qui porte le nom du sélecteur permet d'écrire `[appTypeColor]="type"`.
- La directive définit une **variable CSS** : les styles des éléments enfants s'en servent avec `var(--type-color)`.

### 3. `hostDirectives`

Un composant peut appliquer une directive à son propre élément hôte, et exposer ses entrées sous un autre nom :

```ts
@Component({
  selector: 'app-type-badge',
  hostDirectives: [{ directive: TypeColor, inputs: ['appTypeColor: type'] }],
  …
})
```

On écrit alors `<app-type-badge [type]="'data'" />` et la directive reçoit la valeur.

### 4. Les pipes

```ts
@Pipe({ name: 'dexNumber' })
export class DexNumberPipe implements PipeTransform {
  transform(id: number): string {
    return formatDexNumber(id);
  }
}
```

- Un pipe est **pur** par défaut : Angular ne le réexécute que si sa valeur d'entrée change.
- Le pipe délègue le calcul à une fonction du domaine, testable sans Angular.
- On l'ajoute aux `imports` du composant qui l'utilise.

Pipes intégrés courants (`@angular/common`) :

| Pipe | Exemple | Résultat (locale `fr`) |
|---|---|---|
| `number` | `{{ 1234.5 \| number }}` | 1 234,5 |
| `percent` | `{{ 0.42 \| percent }}` | 42 % |
| `date` | `{{ today \| date: 'EEEE d MMMM' }}` | dimanche 20 septembre |
| `currency` | `{{ 9.9 \| currency: 'EUR' }}` | 9,90 € |
| `uppercase`, `lowercase` | `{{ 'dev' \| uppercase }}` | DEV |
| `json` | `{{ dev \| json }}` | l'objet en JSON (débogage) |

La locale se configure une fois, dans `app.config.ts`.

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 06.

### Étape 1 — La directive `TypeColor`

`npx ng g directive shared/directives/type-color --skip-tests`, puis remplacez le contenu de `src/app/shared/directives/type-color.ts` :

`src/app/shared/directives/type-color.ts`

```ts
import { Directive, computed, input } from '@angular/core';
import { DevType } from '../../domain/dev.model';

const TYPE_COLORS: Record<DevType, string> = {
  frontend: '#d9480f',
  backend: '#1864ab',
  devops: '#2b8a3e',
  data: '#862e9c',
  mobile: '#c2255c',
  securite: '#495057',
};

/**
 * Directive d'attribut : applique la couleur d'un type à l'élément hôte,
 * via la variable CSS --type-color.
 * Usage : <span [appTypeColor]="'data'">Data</span>
 */
@Directive({
  selector: '[appTypeColor]',
  host: {
    '[style.--type-color]': 'color()',
    '[attr.data-type]': 'appTypeColor()',
  },
})
export class TypeColor {
  readonly appTypeColor = input.required<DevType>();

  protected readonly color = computed(() => TYPE_COLORS[this.appTypeColor()]);
}
```

### Étape 2 — La pastille de type

Créez `src/app/shared/ui/type-badge.ts` :

`src/app/shared/ui/type-badge.ts`

```ts
import { Component, input } from '@angular/core';
import { DevType } from '../../domain/dev.model';
import { TypeColor } from '../directives/type-color';

/** Pastille colorée affichant un type. La directive TypeColor est appliquée à l'hôte. */
@Component({
  selector: 'app-type-badge',
  hostDirectives: [{ directive: TypeColor, inputs: ['appTypeColor: type'] }],
  template: `{{ type() }}`,
  styles: `
    :host {
      display: inline-block;
      padding: 0.1rem 0.6rem;
      border-radius: 999px;
      background: var(--type-color);
      color: #fff;
      font-size: 0.8rem;
      font-weight: 600;
    }
  `,
})
export class TypeBadge {
  readonly type = input.required<DevType>();
}
```

Le composant n'a aucune logique de couleur : il la reçoit de la directive, appliquée à son élément hôte.

### Étape 3 — Le pipe `dexNumber`

`npx ng g pipe shared/pipes/dex-number --skip-tests`, puis remplacez le contenu de `src/app/shared/pipes/dex-number-pipe.ts` :

`src/app/shared/pipes/dex-number-pipe.ts`

```ts
import { Pipe, PipeTransform } from '@angular/core';
import { formatDexNumber } from '../../domain/dev-rules';

/** Affiche un numéro façon pokédex : {{ 7 | dexNumber }} → #007 */
@Pipe({ name: 'dexNumber' })
export class DexNumberPipe implements PipeTransform {
  transform(id: number): string {
    return formatDexNumber(id);
  }
}
```

### Étape 4 — La carte définitive

La carte passe à des fichiers séparés pour le template et les styles. Remplacez `src/app/features/dex/dev-card.ts` et créez `dev-card.html` et `dev-card.css` dans le même dossier :

`src/app/features/dex/dev-card.ts`

```ts
import { Component, computed, input, output } from '@angular/core';
import { Dev } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';
import { DexNumberPipe } from '../../shared/pipes/dex-number-pipe';
import { TypeBadge } from '../../shared/ui/type-badge';

@Component({
  selector: 'app-dev-card',
  imports: [DexNumberPipe, TypeBadge],
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
  <div class="body">
    <p class="number">{{ dev().id | dexNumber }}</p>
    <h3>{{ dev().name }}</h3>
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

`src/app/features/dex/dev-card.css`

```css
.card {
  display: grid;
  grid-template-columns: auto 1fr;
  gap: 0.75rem;
  height: 100%;
  box-sizing: border-box;
  padding: 1rem;
  border: 1px solid var(--border);
  border-radius: 0.75rem;
}
.card.in-team {
  border-color: var(--accent);
  box-shadow: 0 0 0 1px var(--accent);
}
.body > * {
  margin: 0;
}
.number {
  color: var(--muted);
  font-variant-numeric: tabular-nums;
}
h3 a {
  color: inherit;
}
.title {
  font-size: 0.9rem;
}
.types {
  display: flex;
  gap: 0.25rem;
  margin-block: 0.4rem;
}
.total {
  color: var(--muted);
  font-size: 0.875rem;
}
button {
  grid-column: 1 / -1;
  align-self: end;
}
```

Supprimez `src/app/features/dex/rank-label.ts` : la carte ne l'utilise plus.

Dans `src/app/features/dex/dex-page.ts`, passez la nouvelle entrée à la carte :

`src/app/features/dex/dex-page.ts`

```ts
import { Component, inject } from '@angular/core';
import { DevRepository } from '../../core/data/dev-repository';
import { Team } from '../../core/team/team';
import { EmptyState } from '../../shared/ui/empty-state';
import { DevCard } from './dev-card';

@Component({
  selector: 'app-dex-page',
  imports: [DevCard, EmptyState],
  template: `
    <h1>Pokédex</h1>

    @if (repository.isLoading()) {
      <p role="status">Chargement du pokédex…</p>
    } @else if (repository.error()) {
      <app-empty-state>
        <p class="error">Impossible de charger le pokédex.</p>
        <button actions type="button" (click)="repository.reload()">Réessayer</button>
      </app-empty-state>
    } @else {
      <p>{{ repository.devs().length }} dev(s) · {{ team.size() }} dans l'équipe</p>
      <ul class="grid">
        @for (dev of repository.devs(); track dev.id) {
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
  protected readonly repository = inject(DevRepository);
  protected readonly team = inject(Team);
}
```

Résultat attendu :
- des numéros au format `#001` … `#021` ;
- des pastilles colorées selon le type (orange pour `frontend`, bleu pour `backend`…), qui affichent encore le nom technique du type ;
- après 6 ajouts, les boutons « Ajouter à l'équipe » des autres cartes sont désactivés ; « Retirer de l'équipe » reste possible.

---

## Vous faites

**Exercice 1 — Le pipe `typeLabel`.** Créez un pipe qui transforme un `DevType` en libellé français (`'securite'` → « Sécurité ») à partir de `TYPE_LABELS`. Utilisez-le dans `TypeBadge`.

<details>
<summary>Correction</summary>

`src/app/shared/pipes/type-label-pipe.ts`

```ts
import { Pipe, PipeTransform } from '@angular/core';
import { DevType } from '../../domain/dev.model';
import { TYPE_LABELS } from '../../domain/labels';

/** Libellé lisible d'un type : {{ 'securite' | typeLabel }} → Sécurité */
@Pipe({ name: 'typeLabel' })
export class TypeLabelPipe implements PipeTransform {
  transform(type: DevType): string {
    return TYPE_LABELS[type];
  }
}
```

`src/app/shared/ui/type-badge.ts`

```ts
import { Component, input } from '@angular/core';
import { DevType } from '../../domain/dev.model';
import { TypeColor } from '../directives/type-color';
import { TypeLabelPipe } from '../pipes/type-label-pipe';

/** Pastille colorée affichant un type. La directive TypeColor est appliquée à l'hôte. */
@Component({
  selector: 'app-type-badge',
  imports: [TypeLabelPipe],
  hostDirectives: [{ directive: TypeColor, inputs: ['appTypeColor: type'] }],
  template: `{{ type() | typeLabel }}`,
  styles: `
    :host {
      display: inline-block;
      padding: 0.1rem 0.6rem;
      border-radius: 999px;
      background: var(--type-color);
      color: #fff;
      font-size: 0.8rem;
      font-weight: 600;
    }
  `,
})
export class TypeBadge {
  readonly type = input.required<DevType>();
}
```

Résultat : « Front-end », « Back-end », « Sécurité »…
</details>

**Exercice 2 — L'avatar.** Créez `src/app/shared/ui/dev-avatar.ts`, un composant `DevAvatar` :
- entrées : `name` et `type` ;
- affichage : les initiales (au plus deux, en majuscules ; « Pare-Feulin » → « PF ») dans un disque de la couleur du type ;
- la couleur vient de `TypeColor`, appliquée par `hostDirectives` ;
- l'hôte porte `role="img"` et un `aria-label` « Avatar de … » ;
- la taille se règle avec la variable CSS `--avatar-size` (3,5 rem par défaut).

Affichez-le en tête de la carte.

<details>
<summary>Correction</summary>

`src/app/shared/ui/dev-avatar.ts`

```ts
import { Component, computed, input } from '@angular/core';
import { DevType } from '../../domain/dev.model';
import { TypeColor } from '../directives/type-color';

/** Avatar généré : initiales du dev sur la couleur de son type principal. */
@Component({
  selector: 'app-dev-avatar',
  hostDirectives: [{ directive: TypeColor, inputs: ['appTypeColor: type'] }],
  host: { role: 'img', '[attr.aria-label]': '"Avatar de " + name()' },
  template: `{{ initials() }}`,
  styles: `
    :host {
      display: grid;
      place-items: center;
      width: var(--avatar-size, 3.5rem);
      aspect-ratio: 1;
      border-radius: 50%;
      background: var(--type-color);
      color: #fff;
      font-weight: 700;
      font-size: calc(var(--avatar-size, 3.5rem) / 2.6);
    }
  `,
})
export class DevAvatar {
  readonly name = input.required<string>();
  readonly type = input.required<DevType>();

  protected readonly initials = computed(() =>
    this.name()
      .split(/[\s-]+/)
      .filter(Boolean)
      .slice(0, 2)
      .map((part) => part[0].toUpperCase())
      .join(''),
  );
}
```

`src/app/features/dex/dev-card.ts`

```ts
import { Component, computed, input, output } from '@angular/core';
import { Dev } from '../../domain/dev.model';
import { totalStats } from '../../domain/dev-rules';
import { DexNumberPipe } from '../../shared/pipes/dex-number-pipe';
import { DevAvatar } from '../../shared/ui/dev-avatar';
import { TypeBadge } from '../../shared/ui/type-badge';

@Component({
  selector: 'app-dev-card',
  imports: [DexNumberPipe, DevAvatar, TypeBadge],
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
    <h3>{{ dev().name }}</h3>
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

Résultat : un disque orange « S » pour Stagiairon, gris « PF » pour Pare-Feulin.
</details>

**Exercice 3 — La locale française.** Dans `app.config.ts`, enregistrez la locale française et fournissez `LOCALE_ID`.

<details>
<summary>Correction</summary>

`src/app/app.config.ts`

```ts
import { ApplicationConfig, LOCALE_ID, provideBrowserGlobalErrorListeners } from '@angular/core';
import { registerLocaleData } from '@angular/common';
import localeFr from '@angular/common/locales/fr';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';
import { loadingInterceptor } from './core/http/loading-interceptor';

registerLocaleData(localeFr);

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes),
    provideHttpClient(withInterceptors([loadingInterceptor])),
    { provide: LOCALE_ID, useValue: 'fr' },
  ],
};
```

Vérification : ajoutez temporairement `DecimalPipe` et `PercentPipe` (de `@angular/common`) aux `imports` de `DexPage`, et `<p>{{ 1234.5 | number }} · {{ 0.42 | percent }}</p>` dans son template. La page affiche « 1 234,5 · 42 % » ; sans la locale, elle afficherait « 1,234.5 · 42% ». Retirez ensuite ces ajouts.
</details>

---

## Check-list

- [ ] Je sais écrire une directive d'attribut avec `host`.
- [ ] Je sais appliquer une directive à un composant avec `hostDirectives`.
- [ ] Je sais écrire un pipe et utiliser les pipes intégrés.
- [ ] La locale française est configurée.

---

[← 06 · Services, injection et HTTP](06-services-http.md) · [Sommaire](README.md) · [08 · Routing →](08-routing.md)
