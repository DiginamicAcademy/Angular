[← 05 · Composants et signaux](05-composants-signaux.md) · [Sommaire](README.md)

---

# Phase 05 — Services, injection de dépendances et HTTP

## Objectifs

- Créer un service avec `@Service()` et l'injecter avec `inject()`.
- Partager un état entre composants grâce à un service.
- Charger des données avec `httpResource` et gérer les états de chargement et d'erreur.
- Valider une réponse HTTP à l'exécution.
- Écrire un intercepteur HTTP fonctionnel.
- Rendre une valeur configurable avec un `InjectionToken`.

Point d'arrivée : le pokédex complet chargé depuis un fichier JSON, et une équipe partagée entre composants.

---

## On théorise

### 1. Services et injection de dépendances

Un **service** est une classe qui porte une responsabilité hors de l'affichage : accès aux données, état partagé, logique technique.

```ts
@Service()                    // un seul exemplaire (singleton) pour toute l'application
export class Team { … }

export class DexPage {
  private readonly team = inject(Team); // Angular fournit l'instance
}
```

- Le composant **demande** une dépendance ; il ne la crée pas avec `new`. On peut donc la remplacer, en particulier dans les tests.
- `inject()` ne fonctionne que dans un **contexte d'injection** : initialisation d'un champ, constructeur, fonction de fabrique. Pas dans une méthode appelée plus tard.
- `@Service()` (Angular 22) remplace `@Injectable({ providedIn: 'root' })`, que vous verrez dans du code existant.

Un `InjectionToken` permet d'injecter autre chose qu'une classe, par exemple une URL :

```ts
export const DEVS_URL = new InjectionToken<string>('DEVS_URL', { factory: () => 'data/devs.json' });
const url = inject(DEVS_URL);
```

### 2. `httpResource`

`httpResource` déclenche une requête GET et expose le résultat sous forme de **signaux** :

```ts
const devs = httpResource(() => 'data/devs.json', { parse: parseDevs, defaultValue: [] });

devs.value();      // les données (ou defaultValue)
devs.isLoading();  // true pendant la requête
devs.error();      // l'erreur éventuelle
devs.hasValue();   // true si une valeur est disponible
devs.reload();     // relance la requête
```

- La fonction passée en premier argument est **réactive** : si elle lit un signal et que ce signal change, la requête est relancée. Si elle renvoie `undefined`, aucune requête n'est envoyée.
- L'option `parse` reçoit la réponse brute (`unknown`) et renvoie une valeur typée. C'est l'endroit où l'on **valide** les données. Si `parse` lève une erreur, la ressource passe en état d'erreur.
- **Lire `value()` sur une ressource en erreur lève une exception.** On teste `hasValue()` avant, dans un `computed`.
- `HttpClient` est disponible par défaut. `provideHttpClient()` ne sert qu'à ajouter des options, comme les intercepteurs.

### 3. Les intercepteurs

Un intercepteur s'exécute pour **chaque** requête : ajout d'en-têtes, journalisation, indicateur de chargement, nouvelle tentative.

```
composant ──▶ httpResource ──▶ intercepteur 1 ──▶ intercepteur 2 ──▶ serveur
                                     ◀────────── réponse ───────────┘
```

```ts
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  // avant la requête
  return next(req).pipe(finalize(() => { /* après, succès ou erreur */ }));
};
```

`next(req)` renvoie un `Observable` RxJS. C'est l'un des rares endroits où RxJS reste nécessaire : `pipe` et `finalize` suffisent ici.

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 05.

### Étape 1 — Les données

1. Créez le dossier `public/data/` et le fichier `public/data/devs.json` ci-dessous. Tout fichier de `public/` est servi tel quel : ouvrez http://localhost:4200/data/devs.json, la liste des 21 devs s'affiche.

<details>
<summary>Contenu de devs.json</summary>

`public/data/devs.json`

```json
[
  {
    "id": 1,
    "name": "Stagiairon",
    "title": "Stagiaire front-end",
    "types": [
      "frontend"
    ],
    "stats": {
      "code": 35,
      "debug": 20,
      "archi": 10,
      "tests": 15,
      "communication": 40,
      "cafe": 60
    },
    "languages": [
      "HTML",
      "CSS"
    ],
    "catchphrase": "Ça marche sur ma machine.",
    "evolvesTo": 2
  },
  {
    "id": 2,
    "name": "Juniorax",
    "title": "Développeur front-end junior",
    "types": [
      "frontend"
    ],
    "stats": {
      "code": 55,
      "debug": 40,
      "archi": 25,
      "tests": 35,
      "communication": 50,
      "cafe": 70
    },
    "languages": [
      "HTML",
      "CSS",
      "TypeScript"
    ],
    "catchphrase": "J'ai trouvé la réponse sur un forum.",
    "evolvesTo": 3
  },
  {
    "id": 3,
    "name": "Seniorgon",
    "title": "Développeur front-end senior",
    "types": [
      "frontend"
    ],
    "stats": {
      "code": 80,
      "debug": 75,
      "archi": 65,
      "tests": 70,
      "communication": 60,
      "cafe": 65
    },
    "languages": [
      "TypeScript",
      "Angular",
      "CSS"
    ],
    "catchphrase": "Ça dépend."
  },
  {
    "id": 4,
    "name": "Scriptilou",
    "title": "Stagiaire back-end",
    "types": [
      "backend"
    ],
    "stats": {
      "code": 40,
      "debug": 25,
      "archi": 15,
      "tests": 10,
      "communication": 30,
      "cafe": 55
    },
    "languages": [
      "PHP"
    ],
    "catchphrase": "Pourquoi il y a une table users2 ?",
    "evolvesTo": 5
  },
  {
    "id": 5,
    "name": "Apirex",
    "title": "Développeur API",
    "types": [
      "backend"
    ],
    "stats": {
      "code": 60,
      "debug": 50,
      "archi": 40,
      "tests": 45,
      "communication": 40,
      "cafe": 65
    },
    "languages": [
      "Java",
      "SQL"
    ],
    "catchphrase": "Tout est une ressource REST.",
    "evolvesTo": 6
  },
  {
    "id": 6,
    "name": "Monolitor",
    "title": "Architecte back-end",
    "types": [
      "backend"
    ],
    "stats": {
      "code": 75,
      "debug": 70,
      "archi": 85,
      "tests": 65,
      "communication": 55,
      "cafe": 60
    },
    "languages": [
      "Java",
      "Kotlin",
      "SQL"
    ],
    "catchphrase": "On découpera en microservices plus tard."
  },
  {
    "id": 7,
    "name": "Yamlou",
    "title": "Stagiaire DevOps",
    "types": [
      "devops"
    ],
    "stats": {
      "code": 30,
      "debug": 35,
      "archi": 20,
      "tests": 15,
      "communication": 35,
      "cafe": 60
    },
    "languages": [
      "YAML",
      "Bash"
    ],
    "catchphrase": "Il manque un espace ligne 42.",
    "evolvesTo": 8
  },
  {
    "id": 8,
    "name": "Kubernaute",
    "title": "Ingénieur DevOps",
    "types": [
      "devops"
    ],
    "stats": {
      "code": 55,
      "debug": 60,
      "archi": 50,
      "tests": 40,
      "communication": 40,
      "cafe": 65
    },
    "languages": [
      "YAML",
      "Go",
      "Bash"
    ],
    "catchphrase": "Redémarre le pod.",
    "evolvesTo": 9
  },
  {
    "id": 9,
    "name": "Clustarque",
    "title": "Ingénieur plateforme",
    "types": [
      "devops"
    ],
    "stats": {
      "code": 70,
      "debug": 80,
      "archi": 80,
      "tests": 60,
      "communication": 50,
      "cafe": 60
    },
    "languages": [
      "Go",
      "Terraform",
      "Bash"
    ],
    "catchphrase": "La prod, c'est mon jardin."
  },
  {
    "id": 10,
    "name": "Tabulis",
    "title": "Analyste de données",
    "types": [
      "data"
    ],
    "stats": {
      "code": 40,
      "debug": 35,
      "archi": 30,
      "tests": 30,
      "communication": 55,
      "cafe": 50
    },
    "languages": [
      "SQL",
      "Python"
    ],
    "catchphrase": "Tu as regardé la médiane ?",
    "evolvesTo": 11
  },
  {
    "id": 11,
    "name": "Requêtor",
    "title": "Ingénieur data",
    "types": [
      "data",
      "backend"
    ],
    "stats": {
      "code": 65,
      "debug": 60,
      "archi": 60,
      "tests": 50,
      "communication": 45,
      "cafe": 55
    },
    "languages": [
      "SQL",
      "Python",
      "Scala"
    ],
    "catchphrase": "Ajoute un index."
  },
  {
    "id": 12,
    "name": "Swipio",
    "title": "Développeur mobile",
    "types": [
      "mobile"
    ],
    "stats": {
      "code": 60,
      "debug": 50,
      "archi": 40,
      "tests": 35,
      "communication": 50,
      "cafe": 60
    },
    "languages": [
      "Kotlin",
      "Swift"
    ],
    "catchphrase": "Sur Android, ça passe."
  },
  {
    "id": 13,
    "name": "Notifox",
    "title": "Développeur mobile hybride",
    "types": [
      "mobile",
      "frontend"
    ],
    "stats": {
      "code": 55,
      "debug": 45,
      "archi": 45,
      "tests": 40,
      "communication": 60,
      "cafe": 55
    },
    "languages": [
      "TypeScript",
      "Dart"
    ],
    "catchphrase": "Un seul code, deux stores."
  },
  {
    "id": 14,
    "name": "Chiffrinou",
    "title": "Analyste sécurité",
    "types": [
      "securite"
    ],
    "stats": {
      "code": 45,
      "debug": 55,
      "archi": 40,
      "tests": 45,
      "communication": 45,
      "cafe": 50
    },
    "languages": [
      "Python",
      "Bash"
    ],
    "catchphrase": "Ton mot de passe est dans le dépôt.",
    "evolvesTo": 15
  },
  {
    "id": 15,
    "name": "Pare-Feulin",
    "title": "Ingénieur sécurité",
    "types": [
      "securite",
      "devops"
    ],
    "stats": {
      "code": 60,
      "debug": 70,
      "archi": 65,
      "tests": 55,
      "communication": 40,
      "cafe": 55
    },
    "languages": [
      "Go",
      "Bash",
      "Rust"
    ],
    "catchphrase": "Tout est fermé par défaut."
  },
  {
    "id": 16,
    "name": "Pixelle",
    "title": "Intégratrice",
    "types": [
      "frontend"
    ],
    "stats": {
      "code": 55,
      "debug": 40,
      "archi": 30,
      "tests": 35,
      "communication": 65,
      "cafe": 50
    },
    "languages": [
      "HTML",
      "CSS"
    ],
    "catchphrase": "Il manque deux pixels à gauche."
  },
  {
    "id": 17,
    "name": "Nullos",
    "title": "Développeur back-end",
    "types": [
      "backend"
    ],
    "stats": {
      "code": 50,
      "debug": 70,
      "archi": 35,
      "tests": 40,
      "communication": 35,
      "cafe": 70
    },
    "languages": [
      "Java",
      "C#"
    ],
    "catchphrase": "Cannot read properties of null."
  },
  {
    "id": 18,
    "name": "Commitor",
    "title": "Gardien du dépôt",
    "types": [
      "devops"
    ],
    "stats": {
      "code": 50,
      "debug": 55,
      "archi": 45,
      "tests": 50,
      "communication": 60,
      "cafe": 55
    },
    "languages": [
      "Git",
      "Bash"
    ],
    "catchphrase": "Un commit, une intention."
  },
  {
    "id": 19,
    "name": "Mergeon",
    "title": "Responsable intégration continue",
    "types": [
      "devops",
      "backend"
    ],
    "stats": {
      "code": 55,
      "debug": 60,
      "archi": 55,
      "tests": 75,
      "communication": 45,
      "cafe": 50
    },
    "languages": [
      "YAML",
      "Java"
    ],
    "catchphrase": "La CI est rouge."
  },
  {
    "id": 20,
    "name": "Datalga",
    "title": "Data scientist",
    "types": [
      "data"
    ],
    "stats": {
      "code": 60,
      "debug": 45,
      "archi": 45,
      "tests": 40,
      "communication": 50,
      "cafe": 65
    },
    "languages": [
      "Python",
      "R"
    ],
    "catchphrase": "Corrélation n'est pas causalité."
  },
  {
    "id": 21,
    "name": "Captchat",
    "title": "Développeur sécurité front",
    "types": [
      "securite",
      "frontend"
    ],
    "stats": {
      "code": 55,
      "debug": 50,
      "archi": 40,
      "tests": 50,
      "communication": 45,
      "cafe": 60
    },
    "languages": [
      "TypeScript",
      "CSS"
    ],
    "catchphrase": "Prouvez que vous n'êtes pas un robot."
  }
]
```
</details>

2. Supprimez `src/app/features/dex/mock-devs.ts`.
3. Créez `src/app/core/data/dev-validation.ts` : c'est le code de l'exercice 3 de la phase 02, avec `export`.

`src/app/core/data/dev-validation.ts`

```ts
import { DEV_TYPES, Dev, DevType, STAT_KEYS } from '../../domain/dev.model';

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null;
}

export function isDevType(value: unknown): value is DevType {
  return typeof value === 'string' && (DEV_TYPES as readonly string[]).includes(value);
}

/** Vérifie à l'exécution qu'une valeur a bien la forme d'un Dev. */
export function isDev(value: unknown): value is Dev {
  if (!isRecord(value) || !isRecord(value['stats'])) {
    return false;
  }
  const stats = value['stats'];
  const types = value['types'];
  const languages = value['languages'];
  return (
    typeof value['id'] === 'number' &&
    typeof value['name'] === 'string' &&
    typeof value['title'] === 'string' &&
    typeof value['catchphrase'] === 'string' &&
    Array.isArray(types) &&
    types.length >= 1 &&
    types.length <= 2 &&
    types.every(isDevType) &&
    Array.isArray(languages) &&
    languages.every((language) => typeof language === 'string') &&
    STAT_KEYS.every((key) => typeof stats[key] === 'number') &&
    (value['evolvesTo'] === undefined || typeof value['evolvesTo'] === 'number')
  );
}

/** Valide une réponse brute ; lève une erreur si elle n'est pas une liste de devs. */
export function parseDevs(raw: unknown): Dev[] {
  if (!Array.isArray(raw)) {
    throw new Error('Réponse inattendue : une liste de devs était attendue.');
  }
  const invalid = raw.findIndex((item) => !isDev(item));
  if (invalid !== -1) {
    throw new Error(`Réponse inattendue : l'élément ${invalid} n'est pas un dev valide.`);
  }
  return raw as Dev[];
}
```

### Étape 2 — Le dépôt de données

`npx ng g service core/data/dev-repository --skip-tests`, puis remplacez le contenu de `src/app/core/data/dev-repository.ts` :

`src/app/core/data/dev-repository.ts`

```ts
import { Service, computed } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { parseDevs } from './dev-validation';

/** Accès aux devs du pokédex. */
@Service()
export class DevRepository {
  private readonly remote = httpResource(() => 'data/devs.json', {
    parse: parseDevs,
    defaultValue: [],
  });

  /** Lire value() d'une ressource en erreur lève une exception : on vérifie hasValue() avant. */
  readonly devs = computed(() => (this.remote.hasValue() ? this.remote.value() : []));
  readonly isLoading = this.remote.isLoading;
  readonly error = this.remote.error;

  byId(id: number) {
    return this.devs().find((dev) => dev.id === id);
  }

  reload(): void {
    this.remote.reload();
  }
}
```

- l'URL est relative (`data/devs.json`, sans `/` initial) : elle fonctionnera aussi quand l'application sera publiée dans un sous-dossier ;
- `parse: parseDevs` : une réponse mal formée devient une erreur affichable, et non un plantage plus loin ;
- `devs` teste `hasValue()` avant de lire `value()`.

### Étape 3 — L'équipe partagée

L'équipe était un signal local à la liste. Pour l'afficher aussi dans l'en-tête et sur la fiche d'un dev, elle devient un service. `npx ng g service core/team/team --skip-tests`, puis :

`src/app/core/team/team.ts`

```ts
import { Service, computed, signal } from '@angular/core';
import { MAX_TEAM_SIZE } from '../../domain/dev.model';

/** L'équipe, en mémoire et partagée entre les composants. Elle est perdue au rechargement. */
@Service()
export class Team {
  private readonly ids = signal<number[]>([]);

  readonly size = computed(() => this.ids().length);
  readonly isFull = computed(() => this.size() >= MAX_TEAM_SIZE);

  has(id: number): boolean {
    return this.ids().includes(id);
  }

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
}
```

### Étape 4 — La page utilise les services

Remplacez `src/app/features/dex/dex-page.ts` :

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

### Étape 5 — Le test du composant racine

Supprimez `src/app/app.spec.ts`. `App` affiche maintenant une page qui charge des données par HTTP : un test devrait simuler ces requêtes. On apprend à le faire à la phase 11, où l'on écrira de vrais tests. D'ici là, `npx ng test` affiche « No tests found » : c'est normal.

### Résultat attendu

- « 21 dev(s) · 0 dans l'équipe », puis les 21 cartes.
- Dans l'onglet « Réseau » des outils de développement : une seule requête vers `devs.json`.
- Après un rechargement de la page, l'équipe est vide : elle n'est gardée qu'en mémoire (on la conservera à la phase 09).

Testez les deux autres états :
- **chargement** : dans l'onglet « Réseau », choisissez la limitation « 3G lente » et rechargez. « Chargement du pokédex… » s'affiche quelques instants. Revenez ensuite à « Pas de limitation ».
- **erreur** : renommez `public/data/devs.json` en `devs.bak`, rechargez : « Impossible de charger le pokédex. » et un bouton « Réessayer ». Rétablissez le nom, cliquez sur « Réessayer » : la liste revient.

---

## Vous faites

**Exercice 1 — Un indicateur de chargement global.**
1. Créez un service `Loading` dans `src/app/core/http/loading.ts`, qui compte les requêtes en cours et expose un `computed` `active`.
2. Générez l'intercepteur : `npx ng g interceptor core/http/loading --skip-tests`. Il appelle `start()` avant chaque requête et `stop()` à la fin, qu'elle réussisse ou échoue.
3. Déclarez l'intercepteur dans `app.config.ts`.
4. Dans `App`, affichez une barre en haut de page quand `active()` vaut `true`.

<details>
<summary>Correction</summary>

`src/app/core/http/loading.ts`

```ts
import { Service, computed, signal } from '@angular/core';

/** Compte les requêtes HTTP en cours pour afficher un indicateur global. */
@Service()
export class Loading {
  private readonly pending = signal(0);

  readonly active = computed(() => this.pending() > 0);

  start(): void {
    this.pending.update((count) => count + 1);
  }

  stop(): void {
    this.pending.update((count) => Math.max(0, count - 1));
  }
}
```

`src/app/core/http/loading-interceptor.ts`

```ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { finalize } from 'rxjs';
import { Loading } from './loading';

/** Intercepteur fonctionnel : signale le début et la fin de chaque requête. */
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(Loading);
  loading.start();
  return next(req).pipe(finalize(() => loading.stop()));
};
```

`src/app/app.config.ts`

```ts
import { ApplicationConfig, provideBrowserGlobalErrorListeners } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';
import { loadingInterceptor } from './core/http/loading-interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideBrowserGlobalErrorListeners(),
    provideRouter(routes),
    provideHttpClient(withInterceptors([loadingInterceptor])),
  ],
};
```

`App`, avec aussi l'en-tête de l'exercice 3 :

`src/app/app.ts`

```ts
import { Component, inject } from '@angular/core';
import { Loading } from './core/http/loading';
import { Team } from './core/team/team';
import { DexPage } from './features/dex/dex-page';

@Component({
  selector: 'app-root',
  imports: [DexPage],
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
  <strong class="brand">Pokedev</strong>
  <span>Mon équipe ({{ team.size() }}/6)</span>
</header>
<main>
  <app-dex-page />
</main>
```

`src/app/app.css`

```css
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  border-bottom: 1px solid var(--border);
}
.progress {
  position: fixed;
  inset: 0 0 auto;
  height: 3px;
  background: var(--accent);
}
```

`finalize` s'exécute dans tous les cas : succès, erreur ou annulation. Le compteur ne peut donc pas rester bloqué.

Résultat : avec la limitation « 3G lente », une fine barre bleue apparaît en haut de la page pendant le chargement, puis disparaît.
</details>

**Exercice 2 — Une URL configurable.** Remplacez l'URL écrite en dur dans le dépôt par un `InjectionToken<string>` nommé `DEVS_URL`, avec `data/devs.json` comme valeur par défaut. On doit pouvoir la remplacer dans `app.config.ts` avec `{ provide: DEVS_URL, useValue: 'data/autres-devs.json' }`.

<details>
<summary>Correction</summary>

`src/app/core/data/devs-url.ts`

```ts
import { InjectionToken } from '@angular/core';

/** Adresse de la liste des devs. Un jeton d'injection permet de la remplacer (tests, autre serveur). */
export const DEVS_URL = new InjectionToken<string>('DEVS_URL', {
  factory: () => 'data/devs.json',
});
```

`src/app/core/data/dev-repository.ts`

```ts
import { Service, computed, inject } from '@angular/core';
import { httpResource } from '@angular/common/http';
import { parseDevs } from './dev-validation';
import { DEVS_URL } from './devs-url';

/** Accès aux devs du pokédex. */
@Service()
export class DevRepository {
  private readonly url = inject(DEVS_URL);
  private readonly remote = httpResource(() => this.url, {
    parse: parseDevs,
    defaultValue: [],
  });

  /** Lire value() d'une ressource en erreur lève une exception : on vérifie hasValue() avant. */
  readonly devs = computed(() => (this.remote.hasValue() ? this.remote.value() : []));
  readonly isLoading = this.remote.isLoading;
  readonly error = this.remote.error;

  byId(id: number) {
    return this.devs().find((dev) => dev.id === id);
  }

  reload(): void {
    this.remote.reload();
  }
}
```

Le jeton est injecté dans un champ, puis lu dans la fonction de requête. Appeler `inject(DEVS_URL)` **dans** la fonction de requête échouerait : elle est exécutée plus tard, hors du contexte d'injection.

Vérification : ajoutez temporairement `{ provide: DEVS_URL, useValue: 'data/autres-devs.json' }` aux providers de `app.config.ts`. L'onglet « Réseau » montre une requête vers `autres-devs.json` (en erreur 404) et la page affiche l'état d'erreur. Retirez ensuite la ligne.
</details>

**Exercice 3 — Le compteur d'équipe dans l'en-tête.** Ajoutez dans l'en-tête de `App` « Mon équipe (n/6) ». Vérifiez que le compteur suit les clics faits dans les cartes.

<details>
<summary>Correction</summary>

Voir `app.ts` et `app.html` dans la correction de l'exercice 1 : `App` injecte `Team` et affiche `team.size()`.

`DexPage` et `App` reçoivent **la même instance** de `Team` : c'est ce qui synchronise l'affichage.
</details>

---

## Check-list

- [ ] Je sais créer un service et l'injecter avec `inject()`.
- [ ] Je sais ce qu'est un contexte d'injection.
- [ ] Je sais charger des données avec `httpResource` et afficher les trois états.
- [ ] Je sais valider une réponse avec `parse`.
- [ ] Je sais écrire et déclarer un intercepteur.

---

[← 05 · Composants et signaux](05-composants-signaux.md) · [Sommaire](README.md) · [07 · Directives et pipes →](07-directives-pipes.md)
