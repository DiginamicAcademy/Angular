[← 01 · Introduction aux frameworks](01-introduction-frameworks.md) · [Sommaire](README.md)

---

# Phase 02 — TypeScript

## Objectifs

- Comprendre ce que TypeScript ajoute à JavaScript, et ce qu'il n'ajoute pas.
- Typer des variables, des fonctions et des objets.
- Utiliser les unions, le narrowing, les génériques, `Record` et les classes.
- Valider à l'exécution une donnée venue de l'extérieur avec un *type guard*.

## Outil utilisé

Toute cette phase se fait dans le **playground TypeScript** : https://www.typescriptlang.org/play

- La zone de gauche contient le code TypeScript. Les erreurs de typage y sont soulignées en rouge, et listées dans l'onglet **Errors** à droite.
- Le bouton **Run** exécute le code ; ce qu'affiche `console.log` apparaît dans l'onglet **Logs**.
- L'onglet **.JS** montre le JavaScript produit.

Le playground exécute **un seul fichier**, comme un script. Les mots-clés `export` et `import` n'y fonctionnent pas : l'exécution échoue avec « Unexpected token 'export' ». Les exemples de cette phase n'en contiennent donc pas. Ils apparaîtront à la phase 05, quand le code sera réparti dans les fichiers du projet.

---

## On théorise

### 1. Ce qu'est TypeScript

TypeScript est **JavaScript + des types**. Le compilateur vérifie les types, puis produit du JavaScript en les effaçant.

```
dev.ts  ──compilation──▶  vérification des types  ──▶  dev.js (types effacés)  ──▶ navigateur
```

Les types n'existent **qu'à la compilation**. À l'exécution, il n'en reste rien : si un fichier JSON renvoie autre chose que ce qu'on a déclaré, TypeScript ne le verra pas. C'est pourquoi on valide les données externes avec un *type guard* (section 6).

Le playground, comme Angular, active le **mode strict** : `null` et `undefined` doivent être traités explicitement, et les paramètres doivent être typés.

Chaque exemple ci-dessous est complet : collez-le dans le playground à la place du contenu existant, puis cliquez sur **Run**.

### 2. Types de base et inférence

```ts
const devName = 'Stagiairon'; // type littéral 'Stagiairon'
let level = 1; // number, inféré
level += 1;
let title: string; // annotation nécessaire : pas de valeur initiale
title = 'Stagiaire front-end';
const languages: string[] = ['HTML', 'CSS'];

console.log(devName, level, title, languages.length);
```

Logs : `Stagiairon 2 Stagiaire front-end 2`.

TypeScript **déduit** le type quand il le peut. On annote les paramètres de fonction, les valeurs de retour publiques et les variables sans valeur initiale. Écrire `let level: number = 1` est redondant : dans le projet Angular, le lint le signale (règle `no-inferrable-types`).

### 3. Interfaces, propriétés optionnelles, `readonly`

```ts
interface Profile {
  readonly id: number; // non modifiable après création
  name: string;
  mentor?: string; // optionnelle : string | undefined
}

const profile: Profile = { id: 1, name: 'Stagiairon' };
profile.name = 'Stagiairon le Brave'; // autorisé
console.log(profile);
```

Logs : l'objet, avec le nom modifié. Ajoutez la ligne `profile.id = 2;` : l'onglet **Errors** affiche « Cannot assign to 'id' because it is a read-only property ».

### 4. Unions et narrowing

Une **union de littéraux** restreint les valeurs possibles. Le **narrowing** : après une vérification, TypeScript affine le type.

```ts
type Rank = 'junior' | 'confirme' | 'senior';

interface Profile {
  readonly id: number;
  name: string;
  rank: Rank;
  mentor?: string;
}

function describeMentor(p: Profile): string {
  if (p.mentor === undefined) {
    return `${p.name} n'a pas de mentor.`;
  }
  return `${p.name} est suivi par ${p.mentor.toUpperCase()}.`; // ici, mentor est forcément une string
}

console.log(describeMentor({ id: 1, name: 'Stagiairon', rank: 'junior' }));
console.log(describeMentor({ id: 2, name: 'Juniorax', rank: 'confirme', mentor: 'Seniorgon' }));
```

Logs : `Stagiairon n'a pas de mentor.` puis `Juniorax est suivi par SENIORGON.`

Supprimez le bloc `if` : l'onglet **Errors** affiche « 'p.mentor' is possibly 'undefined' ».

Un tableau `as const` permet de dériver une union à partir de valeurs. Le tableau sert à l'exécution (afficher les filtres), l'union sert à la compilation, et les deux restent synchronisés :

```ts
const DEV_TYPES = ['frontend', 'backend', 'devops', 'data', 'mobile', 'securite'] as const;
type DevType = (typeof DEV_TYPES)[number]; // 'frontend' | 'backend' | …

const favorite: DevType = 'data';
console.log(DEV_TYPES.length, 'types ; favori :', favorite);
```

Logs : `6 types ; favori : data`. Remplacez `'data'` par `'cobol'` : erreur de compilation.

### 5. Génériques, `Record`, `Omit`

```ts
function firstOrUndefined<T>(items: readonly T[]): T | undefined {
  return items[0];
}

type StatKey = 'code' | 'debug';
const stats: Record<StatKey, number> = { code: 40, debug: 15 }; // une clé manquante = erreur

interface Dev {
  id: number;
  name: string;
}
type NewDev = Omit<Dev, 'id'>; // { name: string }
const draft: NewDev = { name: 'Testeuse' };

console.log(firstOrUndefined(['Stagiairon', 'Juniorax']), firstOrUndefined<number>([]));
console.log(stats.code + stats.debug, draft.name);
```

Logs : `Stagiairon undefined` puis `55 Testeuse`.

- `<T>` : la fonction fonctionne pour n'importe quel type d'élément, et le type du résultat suit celui du tableau.
- `Record<StatKey, number>` : un objet avec exactement ces clés. Supprimez `debug: 15` : erreur.
- `Omit<Dev, 'id'>` construit un type en retirant des propriétés ; `Partial<T>` rend toutes les propriétés optionnelles.

### 6. Type guard

Un *type guard* vérifie une valeur **à l'exécution** et informe le compilateur du résultat. `unknown` oblige à vérifier avant d'utiliser ; `any` désactive toute vérification, on l'évite.

```ts
interface Profile {
  id: number;
  name: string;
}

function isProfile(value: unknown): value is Profile {
  return (
    typeof value === 'object' &&
    value !== null &&
    typeof (value as Profile).id === 'number' &&
    typeof (value as Profile).name === 'string'
  );
}

const good: unknown = JSON.parse('{"id": 2, "name": "Juniorax"}');
const bad: unknown = JSON.parse('{"id": "2"}');

if (isProfile(good)) {
  console.log('Profil valide :', good.name); // good est un Profile ici
}
console.log('Le second est-il valide ?', isProfile(bad));
```

Logs : `Profil valide : Juniorax` puis `Le second est-il valide ? false`.

### 7. Classes

```ts
interface Profile {
  id: number;
  name: string;
}

class Squad {
  private readonly members: Profile[] = [];

  constructor(readonly maxSize: number) {} // déclare et initialise la propriété

  add(member: Profile): boolean {
    if (this.members.length >= this.maxSize) {
      return false;
    }
    this.members.push(member);
    return true;
  }

  get size(): number {
    return this.members.length;
  }
}

const squad = new Squad(2);
console.log(squad.add({ id: 1, name: 'Stagiairon' }), squad.add({ id: 2, name: 'Juniorax' }));
console.log(squad.add({ id: 3, name: 'Seniorgon' }), 'taille :', squad.size);
```

Logs : `true true` puis `false taille : 2`. Angular utilise les classes pour les composants, les services, les directives et les pipes.

---

## On fait ensemble

**Étape 1.** Ouvrez https://www.typescriptlang.org/play, sélectionnez tout le contenu de l'éditeur et remplacez-le par le fichier suivant, qui reprend toutes les notions :

```ts
// Les bases de TypeScript, appliquées au pokédex.
// Cliquez sur « Run » : les résultats s'affichent dans l'onglet « Logs ».

// 1. Types primitifs et inférence
const devName = 'Stagiairon'; // type littéral 'Stagiairon'
let level = 1; // number, inféré
level += 1;
console.log('1.', devName, 'niveau', level);

// 2. Union de littéraux : seules ces valeurs sont acceptées
type Rank = 'junior' | 'confirme' | 'senior';

// 3. Interface : la forme d'un objet
interface Profile {
  readonly id: number;
  name: string;
  rank: Rank;
  mentor?: string; // propriété optionnelle : string | undefined
}

const profile: Profile = { id: 1, name: devName, rank: 'junior' };
console.log('3.', profile);

// 4. Narrowing : TypeScript affine le type après une vérification
function describeMentor(p: Profile): string {
  if (p.mentor === undefined) {
    return `${p.name} n'a pas de mentor.`;
  }
  return `${p.name} est suivi par ${p.mentor.toUpperCase()}.`;
}
console.log('4.', describeMentor(profile));

// 5. Fonction typée avec valeur de retour explicite
function promote(rank: Rank): Rank {
  switch (rank) {
    case 'junior':
      return 'confirme';
    case 'confirme':
    case 'senior':
      return 'senior';
  }
}
console.log('5.', promote('junior'), promote('senior'));

// 6. Générique : la même fonction pour n'importe quel type d'élément
function firstOrUndefined<T>(items: readonly T[]): T | undefined {
  return items[0];
}
console.log('6.', firstOrUndefined<Rank>(['senior', 'junior']), firstOrUndefined<number>([]));

// 7. Record et keyof
type Skill = 'code' | 'tests';
const skills: Record<Skill, number> = { code: 40, tests: 15 };
function skillValue(key: keyof typeof skills): number {
  return skills[key];
}
console.log('7.', skillValue('code'));

// 8. Classe avec membres privés et readonly
class Squad {
  private readonly members: Profile[] = [];

  constructor(readonly maxSize: number) {}

  add(member: Profile): boolean {
    if (this.members.length >= this.maxSize) {
      return false;
    }
    this.members.push(member);
    return true;
  }

  get size(): number {
    return this.members.length;
  }
}
const squad = new Squad(6);
squad.add(profile);
console.log('8.', squad.size, '/', squad.maxSize);

// 9. Type guard : vérifier à l'exécution une donnée venue de l'extérieur
function isProfile(value: unknown): value is Profile {
  return (
    typeof value === 'object' &&
    value !== null &&
    typeof (value as Profile).id === 'number' &&
    typeof (value as Profile).name === 'string'
  );
}
const fromJson: unknown = JSON.parse('{"id": 2, "name": "Juniorax", "rank": "junior"}');
console.log('9.', isProfile(fromJson) ? fromJson.name : 'invalide');
```

**Étape 2.** Cliquez sur **Run**. L'onglet **Logs** doit afficher, dans l'ordre :

```
1. Stagiairon niveau 2
3. { id: 1, name: "Stagiairon", rank: "junior" }   (présentation variable selon le navigateur)
4. Stagiairon n'a pas de mentor.
5. confirme senior
6. senior undefined
7. 40
8. 1 / 6
9. Juniorax
```

**Étape 3.** Ouvrez l'onglet **.JS** : c'est le même code, sans aucune annotation de type. Les interfaces et les types ont disparu.

**Étape 4.** Provoquez chaque erreur ci-dessous, lisez le message dans l'onglet **Errors**, puis annulez la modification (Ctrl+Z) :

| Modification | Message attendu |
|---|---|
| Dans `profile`, remplacez `rank: 'junior'` par `rank: 'expert'` | Type '"expert"' is not assignable to type 'Rank'. |
| Dans `skills`, supprimez `tests: 15` | Property 'tests' is missing in type '{ code: number; }' but required in type 'Record<Skill, number>'. |
| Dans `describeMentor`, supprimez le bloc `if` | 'p.mentor' is possibly 'undefined'. |
| Après la déclaration de `profile`, ajoutez `profile.id = 42;` | Cannot assign to 'id' because it is a read-only property. |

---

## Vous faites

Chaque exercice se fait dans un playground vide. Le code de l'exercice suivant reprend celui du précédent. À la phase 05, ce code sera réparti dans les fichiers du projet.

**Exercice 1 — Le modèle.** Écrivez le modèle d'un dev :
- les types `DevType` et `StatKey`, dérivés de tableaux `as const` ;
- `DevStats`, qui associe un nombre à chaque statistique : `code`, `debug`, `archi`, `tests`, `communication`, `cafe` ;
- l'interface `Dev` : numéro, nom, poste, un ou deux types, statistiques, langages, phrase fétiche, numéro d'évolution optionnel, indicateur `custom` optionnel ;
- les constantes `MAX_STAT = 100`, `MAX_TOTAL = 420`, `MAX_TEAM_SIZE = 6` ;
- l'interface `DevFilter` : un texte de recherche et un type optionnel.

Vérifiez votre modèle en déclarant un dev et un filtre, puis en les affichant avec `console.log`.

<details>
<summary>Correction</summary>

```ts
// Exercice 1 : le modèle d'un dev (un seul fichier dans le playground)

const DEV_TYPES = ['frontend', 'backend', 'devops', 'data', 'mobile', 'securite'] as const;
type DevType = (typeof DEV_TYPES)[number];

const STAT_KEYS = ['code', 'debug', 'archi', 'tests', 'communication', 'cafe'] as const;
type StatKey = (typeof STAT_KEYS)[number];
type DevStats = Record<StatKey, number>;

const MAX_STAT = 100;
const MAX_TOTAL = 420;
const MAX_TEAM_SIZE = 6;

interface Dev {
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

interface DevFilter {
  readonly query: string;
  readonly type?: DevType;
}

// Vérification : un dev conforme au modèle
const stagiairon: Dev = {
  id: 1,
  name: 'Stagiairon',
  title: 'Stagiaire front-end',
  types: ['frontend'],
  stats: { code: 35, debug: 20, archi: 10, tests: 15, communication: 40, cafe: 60 },
  languages: ['HTML', 'CSS'],
  catchphrase: 'Ça marche sur ma machine.',
  evolvesTo: 2,
};
const filter: DevFilter = { query: 'stag', type: 'frontend' };

console.log(stagiairon.name, stagiairon.types, filter.query);
console.log('Limites :', MAX_STAT, MAX_TOTAL, MAX_TEAM_SIZE, '·', STAT_KEYS.length, 'statistiques');
```

Logs : `Stagiairon ["frontend"] stag` puis `Limites : 100 420 6 · 6 statistiques`.
</details>

**Exercice 2 — Des fonctions typées.** À la suite du modèle, écrivez :
- `totalStats(stats: DevStats): number` ;
- `bestStat(stats: DevStats): StatKey`, qui renvoie la première en cas d'égalité ;
- `formatDexNumber(id: number): string` : 7 → `#007` ;
- `filterDevs(devs, filter)` : filtre par type (principal ou secondaire) et par texte (nom, poste, langages), sans tenir compte de la casse ni des accents ;
- `nextId(devs)` : le plus grand numéro + 1.

Indice pour les accents : `text.normalize('NFD').replace(/[\u0300-\u036f]/g, '')` sépare puis supprime les accents.

<details>
<summary>Correction</summary>

```ts
// Exercice 2 : fonctions typées


const DEV_TYPES = ['frontend', 'backend', 'devops', 'data', 'mobile', 'securite'] as const;
type DevType = (typeof DEV_TYPES)[number];

const STAT_KEYS = ['code', 'debug', 'archi', 'tests', 'communication', 'cafe'] as const;
type StatKey = (typeof STAT_KEYS)[number];
type DevStats = Record<StatKey, number>;

const MAX_STAT = 100;
const MAX_TOTAL = 420;
const MAX_TEAM_SIZE = 6;

interface Dev {
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

interface DevFilter {
  readonly query: string;
  readonly type?: DevType;
}

// ---- Fonctions ----

/** Somme des six statistiques. */
function totalStats(stats: DevStats): number {
  return STAT_KEYS.reduce((sum, key) => sum + stats[key], 0);
}

/** Statistique la plus élevée ; en cas d'égalité, la première dans l'ordre de STAT_KEYS. */
function bestStat(stats: DevStats): StatKey {
  return STAT_KEYS.reduce((best, key) => (stats[key] > stats[best] ? key : best));
}

/** Numéro affiché façon pokédex : 7 → « #007 ». */
function formatDexNumber(id: number): string {
  return `#${String(id).padStart(3, '0')}`;
}

/** Normalise une chaîne pour une recherche insensible à la casse et aux accents. */
function normalize(text: string): string {
  return text.normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().trim();
}

function matchesFilter(dev: Dev, filter: DevFilter): boolean {
  if (filter.type && !dev.types.includes(filter.type)) {
    return false;
  }
  const query = normalize(filter.query);
  if (query === '') {
    return true;
  }
  return [dev.name, dev.title, ...dev.languages].some((text) => normalize(text).includes(query));
}

function filterDevs(devs: readonly Dev[], filter: DevFilter): Dev[] {
  return devs.filter((dev) => matchesFilter(dev, filter));
}

/** Numéro libre suivant : le plus grand numéro existant + 1. */
function nextId(devs: readonly Dev[]): number {
  return devs.reduce((max, dev) => Math.max(max, dev.id), 0) + 1;
}

// ---- Vérifications ----
const devs: Dev[] = [
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
    id: 11,
    name: 'Requêtor',
    title: 'Ingénieur data',
    types: ['data', 'backend'],
    stats: { code: 65, debug: 60, archi: 60, tests: 50, communication: 45, cafe: 55 },
    languages: ['SQL', 'Python', 'Scala'],
    catchphrase: 'Ajoute un index.',
  },
];

console.log(totalStats(devs[0].stats)); // 180
console.log(bestStat(devs[0].stats)); // cafe
console.log(formatDexNumber(7), formatDexNumber(151)); // #007 #151
console.log(filterDevs(devs, { query: 'REQUETOR' }).map((dev) => dev.name)); // ["Requêtor"]
console.log(filterDevs(devs, { query: '', type: 'backend' }).length); // 1
console.log(filterDevs(devs, { query: 'html' }).length); // 1
console.log(nextId(devs)); // 12
```

Logs, dans l'ordre : `180`, `cafe`, `#007 #151`, `["Requêtor"]`, `1`, `1`, `12`.
</details>

**Exercice 3 — Valider à la frontière.** À la suite du modèle, écrivez `isDev(value: unknown): value is Dev`, puis `parseDevs(raw: unknown): Dev[]`, qui lève une erreur si la valeur n'est pas un tableau de devs valides. Testez-les avec `JSON.parse` sur un JSON correct, sur un JSON où un dev a un `id` textuel, puis sur un objet qui n'est pas un tableau.

<details>
<summary>Correction</summary>

```ts
// Exercice 3 : valider à la frontière


const DEV_TYPES = ['frontend', 'backend', 'devops', 'data', 'mobile', 'securite'] as const;
type DevType = (typeof DEV_TYPES)[number];

const STAT_KEYS = ['code', 'debug', 'archi', 'tests', 'communication', 'cafe'] as const;
type StatKey = (typeof STAT_KEYS)[number];
type DevStats = Record<StatKey, number>;

const MAX_STAT = 100;
const MAX_TOTAL = 420;
const MAX_TEAM_SIZE = 6;

interface Dev {
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

interface DevFilter {
  readonly query: string;
  readonly type?: DevType;
}

// ---- Validation ----

function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === 'object' && value !== null;
}

function isDevType(value: unknown): value is DevType {
  return typeof value === 'string' && (DEV_TYPES as readonly string[]).includes(value);
}

/** Vérifie à l'exécution qu'une valeur a bien la forme d'un Dev. */
function isDev(value: unknown): value is Dev {
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
function parseDevs(raw: unknown): Dev[] {
  if (!Array.isArray(raw)) {
    throw new Error('Réponse inattendue : une liste de devs était attendue.');
  }
  const invalid = raw.findIndex((item) => !isDev(item));
  if (invalid !== -1) {
    throw new Error(`Réponse inattendue : l'élément ${invalid} n'est pas un dev valide.`);
  }
  return raw as Dev[];
}

// ---- Vérifications ----
const valid = JSON.parse(
  '[{"id":1,"name":"Stagiairon","title":"Stagiaire front-end","types":["frontend"],' +
    '"stats":{"code":35,"debug":20,"archi":10,"tests":15,"communication":40,"cafe":60},' +
    '"languages":["HTML","CSS"],"catchphrase":"Ça marche sur ma machine."}]',
);
const invalid = JSON.parse('[{"id":"1","name":"Stagiairon"}]');

console.log(parseDevs(valid)[0].name); // Stagiairon
try {
  parseDevs(invalid);
} catch (error) {
  console.log((error as Error).message); // Réponse inattendue : l'élément 0 n'est pas un dev valide.
}
try {
  parseDevs({ devs: [] });
} catch (error) {
  console.log((error as Error).message); // Réponse inattendue : une liste de devs était attendue.
}
```

Logs : `Stagiairon`, puis les deux messages d'erreur.
</details>

---

## Check-list

- [ ] Je sais que les types disparaissent à l'exécution.
- [ ] Je sais écrire une interface, une union et un `Record`.
- [ ] Je sais expliquer le narrowing et la différence entre `unknown` et `any`.
- [ ] Je sais écrire un type guard.

---

[← 01 · Introduction aux frameworks](01-introduction-frameworks.md) · [Sommaire](README.md) · [03 · Les outils et la création du projet →](03-environnement-tooling.md)
