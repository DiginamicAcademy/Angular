[← 10 · Formulaires](10-formulaires.md) · [Sommaire](README.md)

---

# Phase 10 — Tests

## Objectifs

- Écrire des tests unitaires avec Vitest : `describe`, `it`, `expect`, `it.each`.
- Tester des fonctions pures, un pipe, une directive, un service, un intercepteur, une garde.
- Simuler des requêtes HTTP avec `HttpTestingController`.
- Tester un composant : entrées, sorties, rendu, interactions.
- Mesurer la couverture et rédiger un plan de tests.

Point d'arrivée : environ 70 tests qui passent, une couverture de 100 % sur le domaine.

---

## On théorise

### 1. Quoi tester

```
            ▲  coût, lenteur
   E2E      │  quelques parcours complets dans un vrai navigateur
 Composants │  un composant rendu, avec ses entrées, ses sorties, ses interactions
 Unitaires  │  une fonction, un service, sans DOM ni réseau : nombreux et rapides
```

Les fonctions pures du domaine sont les meilleures candidates : rapides à tester, et un bug y a un fort impact.

### 2. Vitest

```ts
describe('formatDexNumber', () => {                 // un groupe
  it('complète sur trois chiffres', () => {         // un cas
    expect(formatDexNumber(7)).toBe('#007');        // une vérification
  });

  it.each([
    [42, '#042'],
    [1024, '#1024'],
  ])('%i → %s', (id, expected) => {                 // un cas par ligne du tableau
    expect(formatDexNumber(id)).toBe(expected);
  });
});
```

Vérifications courantes : `toBe` (égalité stricte), `toEqual` (égalité de contenu), `toContain`, `toHaveLength`, `toBeNull`, `toBeUndefined`, `toThrow`, `toBeInstanceOf`.

Espions : `vi.spyOn(objet, 'méthode')` observe les appels ; `.mockReturnValue(x)` ou `.mockResolvedValue(x)` remplace le résultat. `vi.restoreAllMocks()` rétablit les originaux.

Les fichiers de test se nomment `*.spec.ts`, à côté du fichier testé. `npx ng test` les exécute en continu ; `npx ng test --watch=false` une seule fois ; `npx ng test --watch=false --coverage` mesure la couverture.

### 3. `TestBed`

`TestBed` crée un environnement Angular de test : injection, composants, providers.

```ts
TestBed.configureTestingModule({
  providers: [provideHttpClient(), provideHttpClientTesting(), provideRouter([])],
});
const service = TestBed.inject(Loading);                  // un service
const fixture = TestBed.createComponent(DevCard);         // un composant
fixture.componentRef.setInput('dev', dev);                // une entrée
await fixture.whenStable();                               // attendre le rendu
const element = fixture.nativeElement as HTMLElement;     // le DOM rendu
```

### 4. Simuler HTTP

`provideHttpClientTesting()` intercepte les requêtes : aucune ne part sur le réseau. `HttpTestingController` permet d'y répondre.

L'ordre est important avec `httpResource` :

```ts
const fixture = TestBed.createComponent(DexPage);
TestBed.tick();                                             // 1. les effets s'exécutent : la requête part
http.expectOne('data/devs.json').flush(TEST_DEVS);          // 2. on répond
await TestBed.inject(ApplicationRef).whenStable();          // 3. on attend le rendu
```

`whenStable()` appelé **avant** de répondre attendrait indéfiniment : une requête en cours rend l'application « instable ».

`afterEach(() => http.verify())` vérifie qu'aucune requête inattendue n'a été envoyée.

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 10. Lancez `npx ng test` dans un terminal et laissez-le tourner : chaque fichier de test enregistré est exécuté aussitôt.

### Étape 1 — Tester le domaine

Créez `src/app/domain/dev-rules.spec.ts` :

`src/app/domain/dev-rules.spec.ts`

```ts
import { Dev, DevStats } from './dev.model';
import {
  averageStats,
  bestStat,
  filterDevs,
  formatDexNumber,
  nextId,
  previousEvolution,
  teamCoverage,
  totalStats,
} from './dev-rules';

const stats = (value: number, overrides: Partial<DevStats> = {}): DevStats => ({
  code: value,
  debug: value,
  archi: value,
  tests: value,
  communication: value,
  cafe: value,
  ...overrides,
});

const dev = (overrides: Partial<Dev>): Dev => ({
  id: 1,
  name: 'Stagiairon',
  title: 'Stagiaire front',
  types: ['frontend'],
  stats: stats(50),
  languages: ['HTML', 'CSS'],
  catchphrase: 'Ça marche sur ma machine.',
  ...overrides,
});

describe('totalStats', () => {
  it('additionne les six statistiques', () => {
    expect(totalStats(stats(10))).toBe(60);
  });
});

describe('bestStat', () => {
  it('renvoie la statistique la plus élevée', () => {
    expect(bestStat(stats(10, { cafe: 90 }))).toBe('cafe');
  });

  it("renvoie la première en cas d'égalité", () => {
    expect(bestStat(stats(10))).toBe('code');
  });
});

describe('formatDexNumber', () => {
  it.each([
    [7, '#007'],
    [42, '#042'],
    [151, '#151'],
    [1024, '#1024'],
  ])('%i → %s', (id, expected) => {
    expect(formatDexNumber(id)).toBe(expected);
  });
});

describe('filterDevs', () => {
  const devs = [
    dev({ id: 1, name: 'Stagiairon', types: ['frontend'], languages: ['HTML'] }),
    dev({ id: 2, name: 'Requêtor', types: ['data', 'backend'], languages: ['SQL'] }),
    dev({ id: 3, name: 'Kubernaute', types: ['devops'], languages: ['Go'] }),
  ];

  it('ne filtre rien avec un filtre vide', () => {
    expect(filterDevs(devs, { query: '' })).toHaveLength(3);
  });

  it('filtre par type, principal ou secondaire', () => {
    expect(filterDevs(devs, { query: '', type: 'backend' }).map((d) => d.id)).toEqual([2]);
  });

  it('cherche sans tenir compte de la casse ni des accents', () => {
    expect(filterDevs(devs, { query: 'REQUETOR' }).map((d) => d.id)).toEqual([2]);
  });

  it('cherche aussi dans les langages', () => {
    expect(filterDevs(devs, { query: 'go' }).map((d) => d.id)).toEqual([3]);
  });

  it('combine type et texte', () => {
    expect(filterDevs(devs, { query: 'sql', type: 'devops' })).toEqual([]);
  });
});

describe('teamCoverage', () => {
  it("liste les types présents, dans l'ordre de référence", () => {
    const team = [dev({ types: ['data', 'backend'] }), dev({ types: ['frontend'] })];
    expect(teamCoverage(team)).toEqual(['frontend', 'backend', 'data']);
  });
});

describe('averageStats', () => {
  it('renvoie null pour une équipe vide', () => {
    expect(averageStats([])).toBeNull();
  });

  it('calcule une moyenne arrondie', () => {
    const average = averageStats([dev({ stats: stats(10) }), dev({ stats: stats(15) })]);
    expect(average?.code).toBe(13);
  });
});

describe('nextId', () => {
  it('renvoie 1 pour une liste vide', () => {
    expect(nextId([])).toBe(1);
  });

  it('renvoie le plus grand numéro + 1', () => {
    expect(nextId([dev({ id: 4 }), dev({ id: 21 }), dev({ id: 9 })])).toBe(22);
  });
});

describe('previousEvolution', () => {
  it('trouve le dev qui évolue vers le dev donné', () => {
    const junior = dev({ id: 1, evolvesTo: 2 });
    const senior = dev({ id: 2 });
    expect(previousEvolution([junior, senior], senior)).toBe(junior);
    expect(previousEvolution([junior, senior], junior)).toBeUndefined();
  });
});
```

Résultat : `Tests 18 passed (18)`.

Cassez volontairement `formatDexNumber` dans `dev-rules.ts` (remplacez `padStart(3, '0')` par `padStart(2, '0')`) : Vitest indique quels cas échouent, avec la valeur attendue et la valeur obtenue. Rétablissez ensuite le code.

### Étape 2 — Tester la validation

Créez `src/app/core/data/dev-validation.spec.ts` :

`src/app/core/data/dev-validation.spec.ts`

```ts
import { isDev, parseDevs } from './dev-validation';

const valid = {
  id: 1,
  name: 'Stagiairon',
  title: 'Stagiaire front-end',
  types: ['frontend'],
  stats: { code: 35, debug: 20, archi: 10, tests: 15, communication: 40, cafe: 60 },
  languages: ['HTML', 'CSS'],
  catchphrase: 'Ça marche sur ma machine.',
  evolvesTo: 2,
};

describe('isDev', () => {
  it('accepte un dev valide', () => {
    expect(isDev(valid)).toBe(true);
  });

  it.each([
    ['null', null],
    ['un id texte', { ...valid, id: '1' }],
    ['un type inconnu', { ...valid, types: ['cobol'] }],
    ['trois types', { ...valid, types: ['frontend', 'backend', 'data'] }],
    ['une statistique manquante', { ...valid, stats: { code: 1 } }],
    ['un langage non textuel', { ...valid, languages: [42] }],
  ])('refuse %s', (_label, value) => {
    expect(isDev(value)).toBe(false);
  });
});

describe('parseDevs', () => {
  it('renvoie la liste validée', () => {
    expect(parseDevs([valid])).toEqual([valid]);
  });

  it("lève une erreur si la réponse n'est pas un tableau", () => {
    expect(() => parseDevs({ devs: [] })).toThrow(/liste de devs/);
  });

  it("indique l'élément invalide", () => {
    expect(() => parseDevs([valid, { id: 2 }])).toThrow(/élément 1/);
  });
});
```

### Étape 3 — Tester un composant

Créez des données de test partagées, `src/app/testing/dev-fixtures.ts` :

`src/app/testing/dev-fixtures.ts`

```ts
import { Dev } from '../domain/dev.model';

/** Jeu de données réduit pour les tests. */
export const TEST_DEVS: Dev[] = [
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
    id: 2,
    name: 'Juniorax',
    title: 'Développeur front-end junior',
    types: ['frontend'],
    stats: { code: 55, debug: 40, archi: 25, tests: 35, communication: 50, cafe: 70 },
    languages: ['TypeScript'],
    catchphrase: "J'ai trouvé la réponse sur un forum.",
  },
  {
    id: 11,
    name: 'Requêtor',
    title: 'Ingénieur data',
    types: ['data', 'backend'],
    stats: { code: 65, debug: 60, archi: 60, tests: 50, communication: 45, cafe: 55 },
    languages: ['SQL', 'Python'],
    catchphrase: 'Ajoute un index.',
  },
];
```

Puis `src/app/features/dex/dev-card.spec.ts` :

`src/app/features/dex/dev-card.spec.ts`

```ts
import { TestBed } from '@angular/core/testing';
import { provideRouter } from '@angular/router';
import { TEST_DEVS } from '../../testing/dev-fixtures';
import { DevCard } from './dev-card';

describe('DevCard', () => {
  beforeEach(() => TestBed.configureTestingModule({ providers: [provideRouter([])] }));

  async function render(inTeam = false, teamFull = false) {
    const fixture = TestBed.createComponent(DevCard);
    fixture.componentRef.setInput('dev', TEST_DEVS[2]);
    fixture.componentRef.setInput('inTeam', inTeam);
    fixture.componentRef.setInput('teamFull', teamFull);
    await fixture.whenStable();
    return { fixture, element: fixture.nativeElement as HTMLElement };
  }

  it('affiche le numéro, le nom, les types et le total', async () => {
    const { element } = await render();
    const text = element.textContent ?? '';

    expect(text).toContain('#011');
    expect(text).toContain('Requêtor');
    expect(text).toContain('Data');
    expect(text).toContain('Back-end');
    expect(text).toContain('Total : 335');
    expect(element.querySelector('a')?.getAttribute('href')).toBe('/devs/11');
  });

  it('émet le numéro du dev au clic', async () => {
    const { fixture, element } = await render();
    const emitted: number[] = [];
    fixture.componentInstance.teamToggled.subscribe((id) => emitted.push(id));

    element.querySelector('button')!.click();

    expect(emitted).toEqual([11]);
  });

  it("désactive l'ajout quand l'équipe est pleine", async () => {
    const { element } = await render(false, true);
    expect(element.querySelector('button')!.disabled).toBe(true);
  });

  it("permet de retirer un membre même si l'équipe est pleine", async () => {
    const { element } = await render(true, true);
    const button = element.querySelector('button')!;
    expect(button.disabled).toBe(false);
    expect(button.textContent).toContain('Retirer');
  });
});
```

- `provideRouter([])` : la carte contient un `routerLink` ;
- `setInput` alimente les entrées comme le ferait un parent ;
- on s'abonne à la sortie (`subscribe`) pour vérifier ce qu'elle émet.

Résultat : les trois fichiers passent.

---

## Vous faites

**Exercice 1 — Pipes et directive.** Testez `DexNumberPipe`, `TypeLabelPipe` et `TypeColor`. Pour la directive, créez dans le fichier de test un petit composant hôte qui l'utilise avec un signal, puis vérifiez la variable CSS avant et après modification du signal.

<details>
<summary>Correction</summary>

`src/app/shared/pipes/pipes.spec.ts`

```ts
import { DexNumberPipe } from './dex-number-pipe';
import { TypeLabelPipe } from './type-label-pipe';

describe('DexNumberPipe', () => {
  it('complète le numéro sur trois chiffres', () => {
    expect(new DexNumberPipe().transform(7)).toBe('#007');
  });
});

describe('TypeLabelPipe', () => {
  it('renvoie le libellé français', () => {
    expect(new TypeLabelPipe().transform('securite')).toBe('Sécurité');
  });
});
```

`src/app/shared/directives/type-color.spec.ts`

```ts
import { Component, signal } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { DevType } from '../../domain/dev.model';
import { TypeColor } from './type-color';

@Component({
  imports: [TypeColor],
  template: `<span [appTypeColor]="type()">badge</span>`,
})
class Host {
  readonly type = signal<DevType>('data');
}

describe('TypeColor', () => {
  it('applique la couleur du type et la met à jour', async () => {
    const fixture = TestBed.createComponent(Host);
    await fixture.whenStable();
    const span = (fixture.nativeElement as HTMLElement).querySelector('span')!;

    expect(span.style.getPropertyValue('--type-color')).toBe('#862e9c');
    expect(span.dataset['type']).toBe('data');

    fixture.componentInstance.type.set('devops');
    await fixture.whenStable();
    expect(span.style.getPropertyValue('--type-color')).toBe('#2b8a3e');
  });
});
```
</details>

**Exercice 2 — Services et HTTP.** Testez :
- `DevRepository` : chargement, réponse invalide (état d'erreur, liste vide), ajout d'un dev (numéro, indicateur `custom`, sauvegarde) ;
- `Team` : ajout et retrait, refus d'un septième membre, couverture, relecture du stockage, stockage corrompu ;
- `loadingInterceptor` : actif pendant la requête, inactif après un succès ou une erreur.

<details>
<summary>Correction</summary>

`src/app/core/data/dev-repository.spec.ts`

```ts
import { ApplicationRef } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TEST_DEVS } from '../../testing/dev-fixtures';
import { DevRepository } from './dev-repository';

describe('DevRepository', () => {
  let http: HttpTestingController;
  let repository: DevRepository;

  beforeEach(() => {
    localStorage.clear();
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    http = TestBed.inject(HttpTestingController);
    repository = TestBed.inject(DevRepository);
    // Exécute les effets : la ressource envoie alors sa requête.
    TestBed.tick();
  });

  afterEach(() => http.verify());

  it('charge les devs depuis le fichier JSON', async () => {
    http.expectOne('data/devs.json').flush(TEST_DEVS);
    await TestBed.inject(ApplicationRef).whenStable();

    expect(repository.devs()).toHaveLength(3);
    expect(repository.byId(11)?.name).toBe('Requêtor');
  });

  it('passe en erreur si la réponse est invalide', async () => {
    http.expectOne('data/devs.json').flush([{ id: 'x' }]);
    await TestBed.inject(ApplicationRef).whenStable();

    expect(repository.error()).toBeTruthy();
    expect(repository.devs()).toEqual([]);
  });

  it('ajoute un dev personnalisé avec le numéro suivant et le sauvegarde', async () => {
    http.expectOne('data/devs.json').flush(TEST_DEVS);
    await TestBed.inject(ApplicationRef).whenStable();

    const id = repository.add({
      name: 'Testeuse',
      title: 'QA',
      types: ['backend'],
      stats: TEST_DEVS[0].stats,
      languages: ['Java'],
      catchphrase: 'Et si le champ est vide ?',
    });
    TestBed.tick();

    expect(id).toBe(12);
    expect(repository.byId(12)?.custom).toBe(true);
    expect(localStorage.getItem('pokedev.custom-devs.v1')).toContain('Testeuse');
  });
});
```

`src/app/core/team/team.spec.ts`

```ts
import { ApplicationRef } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { TEST_DEVS } from '../../testing/dev-fixtures';
import { Team } from './team';

describe('Team', () => {
  let team: Team;

  async function setup(): Promise<void> {
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting()],
    });
    team = TestBed.inject(Team);
    TestBed.tick();
    TestBed.inject(HttpTestingController).expectOne('data/devs.json').flush(TEST_DEVS);
    await TestBed.inject(ApplicationRef).whenStable();
  }

  beforeEach(() => localStorage.clear());

  it('ajoute puis retire un dev', async () => {
    await setup();
    team.toggle(1);
    expect(team.has(1)).toBe(true);
    expect(team.members().map((dev) => dev.name)).toEqual(['Stagiairon']);

    team.toggle(1);
    expect(team.size()).toBe(0);
  });

  it('refuse un septième membre', async () => {
    await setup();
    for (const id of [1, 2, 3, 4, 5, 6]) {
      team.toggle(id);
    }
    expect(team.isFull()).toBe(true);
    expect(team.toggle(7)).toBe(false);
    expect(team.size()).toBe(6);
  });

  it('calcule la couverture des types', async () => {
    await setup();
    team.toggle(1);
    team.toggle(11);
    expect(team.coverage()).toEqual(['frontend', 'backend', 'data']);
  });

  it("relit l'équipe sauvegardée", async () => {
    localStorage.setItem('pokedev.team.v1', JSON.stringify([2, 11]));
    await setup();
    expect(team.members().map((dev) => dev.id)).toEqual([2, 11]);
  });

  it('ignore un contenu de stockage corrompu', async () => {
    localStorage.setItem('pokedev.team.v1', '{pas du json');
    await setup();
    expect(team.size()).toBe(0);
  });
});
```

`src/app/core/http/loading-interceptor.spec.ts`

```ts
import { TestBed } from '@angular/core/testing';
import { HttpClient, provideHttpClient, withInterceptors } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { Loading } from './loading';
import { loadingInterceptor } from './loading-interceptor';

describe('loadingInterceptor', () => {
  let http: HttpClient;
  let controller: HttpTestingController;
  let loading: Loading;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideHttpClient(withInterceptors([loadingInterceptor])),
        provideHttpClientTesting(),
      ],
    });
    http = TestBed.inject(HttpClient);
    controller = TestBed.inject(HttpTestingController);
    loading = TestBed.inject(Loading);
  });

  afterEach(() => controller.verify());

  it('est actif pendant la requête', () => {
    http.get('/api').subscribe();
    expect(loading.active()).toBe(true);

    controller.expectOne('/api').flush({});
    expect(loading.active()).toBe(false);
  });

  it("redevient inactif après une erreur", () => {
    http.get('/api').subscribe({ error: () => undefined });
    controller.expectOne('/api').flush('Erreur', { status: 500, statusText: 'Server Error' });
    expect(loading.active()).toBe(false);
  });
});
```
</details>

**Exercice 3 — Gardes.** Testez `devIdGuard` (`'7'` accepté ; `'abc'`, `'0'`, `'-3'`, `'2.5'` redirigés) et `unsavedChangesGuard` (pas de question sans modification ; sinon, la réponse de l'utilisateur est suivie). Une garde qui appelle `inject()` s'exécute avec `TestBed.runInInjectionContext`.

<details>
<summary>Correction</summary>

`src/app/core/navigation/guards.spec.ts`

```ts
import { TestBed } from '@angular/core/testing';
import {
  ActivatedRouteSnapshot,
  RouterStateSnapshot,
  UrlTree,
  convertToParamMap,
  provideRouter,
} from '@angular/router';
import { devIdGuard } from './dev-id-guard';
import { HasUnsavedChanges, unsavedChangesGuard } from './unsaved-changes-guard';

function routeWithId(id: string): ActivatedRouteSnapshot {
  return { paramMap: convertToParamMap({ id }) } as ActivatedRouteSnapshot;
}

describe('devIdGuard', () => {
  beforeEach(() => TestBed.configureTestingModule({ providers: [provideRouter([])] }));

  const run = (id: string) =>
    TestBed.runInInjectionContext(() => devIdGuard(routeWithId(id), {} as RouterStateSnapshot));

  it('accepte un numéro entier positif', () => {
    expect(run('7')).toBe(true);
  });

  it.each(['abc', '0', '-3', '2.5'])('redirige « %s » vers la page 404', (id) => {
    const result = run(id);
    expect(result).toBeInstanceOf(UrlTree);
    expect(String(result)).toBe('/introuvable');
  });
});

describe('unsavedChangesGuard', () => {
  const run = (dirty: boolean) =>
    unsavedChangesGuard(
      { hasUnsavedChanges: () => dirty } as HasUnsavedChanges,
      {} as ActivatedRouteSnapshot,
      {} as RouterStateSnapshot,
      {} as RouterStateSnapshot,
    );

  afterEach(() => vi.restoreAllMocks());

  it('laisse partir sans question si rien n’est modifié', () => {
    const confirmSpy = vi.spyOn(window, 'confirm');
    expect(run(false)).toBe(true);
    expect(confirmSpy).not.toHaveBeenCalled();
  });

  it("suit la réponse de l'utilisateur sinon", () => {
    vi.spyOn(window, 'confirm').mockReturnValue(false);
    expect(run(true)).toBe(false);
  });
});
```
</details>

**Exercice 4 — Pages et brouillon.** Testez `dev-draft.ts`, `DexPage` (tous les devs, filtre par type, type inconnu ignoré, recherche après le debounce, ajout à l'équipe) et `CreateDevPage` (invalide au départ, nom déjà pris, types identiques, total dépassé, enregistrement et navigation).

<details>
<summary>Correction</summary>

`src/app/features/create/dev-draft.spec.ts`

```ts
import { draftToDev, emptyDraft, splitLanguages } from './dev-draft';

describe('splitLanguages', () => {
  it('découpe, nettoie et ignore les éléments vides', () => {
    expect(splitLanguages(' TypeScript, , SQL ')).toEqual(['TypeScript', 'SQL']);
  });
});

describe('draftToDev', () => {
  it("n'ajoute pas de type secondaire vide", () => {
    const dev = draftToDev({ ...emptyDraft(), name: ' Testeuse ', primaryType: 'data' });
    expect(dev.name).toBe('Testeuse');
    expect(dev.types).toEqual(['data']);
  });

  it('conserve le type secondaire', () => {
    const dev = draftToDev({ ...emptyDraft(), primaryType: 'data', secondaryType: 'backend' });
    expect(dev.types).toEqual(['data', 'backend']);
  });
});
```

`src/app/features/dex/dex-page.spec.ts`

```ts
import { ApplicationRef } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { provideRouter } from '@angular/router';
import { TEST_DEVS } from '../../testing/dev-fixtures';
import { DexPage } from './dex-page';

describe('DexPage', () => {
  let http: HttpTestingController;

  beforeEach(() => {
    localStorage.clear();
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting(), provideRouter([])],
    });
    http = TestBed.inject(HttpTestingController);
  });

  afterEach(() => http.verify());

  async function render(type?: string) {
    const fixture = TestBed.createComponent(DexPage);
    if (type) {
      fixture.componentRef.setInput('type', type);
    }
    TestBed.tick();
    http.expectOne('data/devs.json').flush(TEST_DEVS);
    await TestBed.inject(ApplicationRef).whenStable();
    return fixture.nativeElement as HTMLElement;
  }

  const names = (element: HTMLElement) =>
    Array.from(element.querySelectorAll('app-dev-card h3')).map((h3) => h3.textContent?.trim());

  it('affiche tous les devs', async () => {
    const element = await render();
    expect(names(element)).toEqual(['Stagiairon', 'Juniorax', 'Requêtor']);
    expect(element.textContent).toContain('3 dev(s)');
  });

  it('filtre selon le paramètre de requête type', async () => {
    const element = await render('backend');
    expect(names(element)).toEqual(['Requêtor']);
  });

  it('ignore un type inconnu', async () => {
    const element = await render('cobol');
    expect(names(element)).toHaveLength(3);
  });

  it('filtre selon la recherche après le debounce', async () => {
    const element = await render();
    const input = element.querySelector<HTMLInputElement>('#search')!;
    input.value = 'junior';
    input.dispatchEvent(new Event('input'));

    await new Promise((resolve) => setTimeout(resolve, 250));
    await TestBed.inject(ApplicationRef).whenStable();

    expect(names(element)).toEqual(['Juniorax']);
  });

  it('ajoute un dev à l’équipe depuis sa carte', async () => {
    const element = await render();
    element.querySelector<HTMLButtonElement>('app-dev-card button')!.click();
    await TestBed.inject(ApplicationRef).whenStable();

    expect(element.querySelector('app-dev-card button')?.textContent).toContain('Retirer');
  });
});
```

`src/app/features/create/create-dev-page.spec.ts`

```ts
import { ApplicationRef } from '@angular/core';
import { TestBed } from '@angular/core/testing';
import { provideHttpClient } from '@angular/common/http';
import { HttpTestingController, provideHttpClientTesting } from '@angular/common/http/testing';
import { Router, provideRouter } from '@angular/router';
import { DevRepository } from '../../core/data/dev-repository';
import { TEST_DEVS } from '../../testing/dev-fixtures';
import { CreateDevPage } from './create-dev-page';

describe('CreateDevPage', () => {
  let element: HTMLElement;
  let component: CreateDevPage;

  beforeEach(async () => {
    localStorage.clear();
    TestBed.configureTestingModule({
      providers: [provideHttpClient(), provideHttpClientTesting(), provideRouter([])],
    });
    const fixture = TestBed.createComponent(CreateDevPage);
    element = fixture.nativeElement;
    component = fixture.componentInstance;
    TestBed.tick();
    TestBed.inject(HttpTestingController).expectOne('data/devs.json').flush(TEST_DEVS);
    await stable();
  });

  const stable = () => TestBed.inject(ApplicationRef).whenStable();

  async function type(selector: string, value: string): Promise<void> {
    const field = element.querySelector<HTMLInputElement | HTMLSelectElement>(selector)!;
    field.value = value;
    field.dispatchEvent(new Event('input'));
    field.dispatchEvent(new Event('change'));
    field.dispatchEvent(new Event('blur'));
    await stable();
  }

  const submitButton = () => element.querySelector<HTMLButtonElement>('button[type=submit]')!;

  it('est invalide au départ', () => {
    expect(submitButton().disabled).toBe(true);
    expect(component.hasUnsavedChanges()).toBe(false);
  });

  it('refuse un nom déjà pris', async () => {
    await type('#name', 'juniorax');
    expect(element.textContent).toContain('Ce nom est déjà pris.');
  });

  it('refuse deux types identiques', async () => {
    await type('#secondaryType', 'frontend');
    expect(element.textContent).toContain('Le type secondaire doit différer du type principal.');
  });

  it('refuse un total supérieur à 420', async () => {
    for (const key of ['code', 'debug', 'archi', 'tests', 'communication']) {
      await type(`#stat-${key}`, '90');
    }
    expect(element.textContent).toContain('Le total ne doit pas dépasser 420.');
  });

  it('enregistre un dev valide et ouvre sa fiche', async () => {
    const navigate = vi.spyOn(TestBed.inject(Router), 'navigate').mockResolvedValue(true);
    await type('#name', 'Testeuse');
    await type('#title', 'Ingénieure qualité');
    await type('#languages', 'Java, Gherkin');
    expect(component.hasUnsavedChanges()).toBe(true);
    expect(submitButton().disabled).toBe(false);

    submitButton().click();
    await stable();

    const created = TestBed.inject(DevRepository).byId(12);
    expect(created?.name).toBe('Testeuse');
    expect(created?.languages).toEqual(['Java', 'Gherkin']);
    expect(navigate).toHaveBeenCalledWith(['/devs', 12]);
    expect(component.hasUnsavedChanges()).toBe(false);
  });
});
```

Pour la recherche, on attend un peu plus que le délai du debounce avant de vérifier. Pour la navigation, on espionne `Router.navigate` : le test vérifie l'appel sans charger la page de destination.

Résultat, avec tous les fichiers : `Test Files 12 passed (12)` et `Tests 65 passed (65)`.
</details>

**Exercice 5 — Couverture et plan de tests.**
1. Lancez `npx ng test --watch=false --coverage` et ouvrez `coverage/pokedev/index.html` dans un navigateur. Repérez un fichier peu couvert. Le dossier `domain` doit être couvert à 100 %.
2. Rédigez le plan de tests à partir des scénarios Gherkin de la phase 04 : chaque scénario donne un cas, avec au moins 8 cas au total, dont 2 tests manuels. Pour chaque cas : identifiant, user story, étapes, résultat attendu, type (automatisé ou manuel), fichier de test.

<details>
<summary>Exemple de plan (extrait)</summary>

| Id | Story | Étapes | Résultat attendu | Type | Test |
|---|---|---|---|---|---|
| T01 | 2 | Ouvrir `/devs?type=backend` | Seuls les devs de type Back-end sont listés | Auto | `dex-page.spec.ts` |
| T02 | 2 | Saisir « junior » dans la recherche | Seul Juniorax est listé | Auto | `dex-page.spec.ts` |
| T03 | 3 | Ouvrir `/devs/abc` | Page introuvable | Auto | `guards.spec.ts` |
| T04 | 4 | Ajouter 6 devs, puis un 7ᵉ | Les boutons « Ajouter » sont désactivés | Auto | `team.spec.ts`, `dev-card.spec.ts` |
| T05 | 5 | Composer une équipe, recharger | L'équipe est conservée | Auto | `team.spec.ts` |
| T06 | 6 | Saisir un nom existant | « Ce nom est déjà pris. » | Auto | `create-dev-page.spec.ts` |
| T07 | 6 | Créer un dev valide | La fiche #022 s'ouvre | Auto | `create-dev-page.spec.ts` |
| T08 | 6 | Saisir un nom puis quitter la page | Demande de confirmation | Manuel | — |
| T09 | 1 | Afficher sur un écran de 360 px de large | Cartes sur une colonne, aucun défilement horizontal | Manuel | — |
</details>

---

## Check-list

- [ ] `npx ng test --watch=false` passe entièrement.
- [ ] Je sais tester une fonction pure, un service et un composant.
- [ ] Je connais l'ordre `tick` → `flush` → `whenStable` avec `httpResource`.
- [ ] Je sais lire un rapport de couverture.
- [ ] Mon plan de tests couvre toutes les user stories.

---

[← 10 · Formulaires](10-formulaires.md) · [Sommaire](README.md) · [12 · Sécurité, build et déploiement →](12-securite-build-deploiement.md)
