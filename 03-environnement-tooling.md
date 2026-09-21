[← 02 · TypeScript](02-typescript.md) · [Sommaire](README.md)

---

# Phase 03 — Les outils et la création du projet

## Objectifs

- Installer et configurer son poste : Node.js, Git, un éditeur et ses extensions, un navigateur et son extension Angular.
- Connaître les commandes de la CLI Angular et savoir ce que chacune produit.
- Connaître les ressources d'aide : documentation, guide de migration, serveur MCP pour les assistants IA.
- Créer le projet Pokedev et comprendre chaque fichier généré.

---

## On théorise

### 1. Les outils du poste

Trois familles d'outils, indépendantes d'Angular, mais indispensables au quotidien.

| Famille | Outil | Ce qu'il apporte |
|---|---|---|
| Exécution | **Node.js** et **npm** | Exécutent la CLI, les tests et le serveur de développement ; installent les dépendances. |
| Versionnement | **Git** | Historique du code, travail à plusieurs, déclenchement de la CI. |
| Éditeur | **VS Code** + *Angular Language Service*, *ESLint*, *Prettier*, *EditorConfig* — ou **WebStorm**, qui intègre tout cela | Autocomplétion et erreurs dans les templates, lint et formatage à l'enregistrement, débogage. |
| Navigateur | **Chrome** ou **Firefox** + *Angular DevTools* | Arbre des composants, valeur des signaux, profilage de la détection de changement. |

L'installation détaillée de chacun est décrite dans la section « Installer les outils », plus bas.

### 2. Les outils Angular : la CLI

Une seule commande, `ng`, appelée avec `npx` pour utiliser la version du projet.

```
               ┌──────────── Angular CLI (ng) ────────────┐
 ng new ──────▶│ crée le workspace                        │
 ng serve ────▶│ build de dev + serveur + rechargement    │──▶ http://localhost:4200
 ng generate ─▶│ génère composants, services, pipes…      │
 ng test ─────▶│ tests unitaires (Vitest)                 │
 ng lint ─────▶│ analyse statique (angular-eslint)        │
 ng build ────▶│ build de production optimisée            │──▶ dist/
 ng add ──────▶│ installe ET configure une bibliothèque   │
 ng update ───▶│ met à jour ET migre le code              │
 ng mcp ──────▶│ serveur MCP pour assistants IA           │
               └──────────────────────────────────────────┘
```

| Commande | Ce qu'elle fait | Options utiles |
|---|---|---|
| `ng new <nom>` | Crée le projet et son premier commit | `--style=css`, `--ssr=false` |
| `ng serve` | Sert l'application et la recharge à chaque enregistrement | `--port`, `--configuration production` |
| `ng generate <type> <chemin>` (`ng g`) | Crée un composant, service, pipe, directive, intercepteur, garde, resolver | `--dry-run`, `--flat`, `--skip-tests`, `--inline-template` |
| `ng test` | Lance Vitest en mode surveillance | `--watch=false`, `--coverage` |
| `ng lint` | Analyse le code et les templates | `--fix` |
| `ng build` | Build de production dans `dist/` | `--base-href`, `--configuration` |
| `ng add <paquet>` | Installe **et configure** une bibliothèque | — |
| `ng update` | Met à jour les dépendances **et migre le code** | `@angular/core @angular/cli` |
| `ng version` | Versions d'Angular, de Node et des paquets installés | — |
| `ng mcp` | Démarre le serveur MCP pour un assistant IA | — |

`npx ng` utilise la CLI installée dans le projet ; `npx @angular/cli@22 new …` télécharge la CLI le temps de créer le projet. Une installation globale (`npm i -g @angular/cli`) n'est pas nécessaire et finit souvent en décalage avec la version du projet.

### 3. Les outils d'aide

| Ressource | Adresse | Usage |
|---|---|---|
| Documentation officielle | https://angular.dev | Référence des API, guides, exemples exécutables |
| Playground | https://angular.dev/playground | Essayer un composant sans rien installer |
| Guide de migration | https://angular.dev/update-guide | Étapes exactes pour passer d'une version majeure à la suivante |
| Guide de style | https://angular.dev/style-guide | Conventions officielles de nommage et d'organisation |
| Serveur MCP de la CLI | `npx ng mcp` | Donne à un assistant IA (Claude, Copilot, Cursor…) l'accès à la documentation et aux bonnes pratiques de **la version installée** |
| Angular AI Tutor | https://angular.dev/ai/ai-tutor | Tutoriel guidé par un assistant |
| Communauté | Discord Angular, Stack Overflow (`angular`) | Questions restées sans réponse |

Deux précautions avec les assistants IA : ils proposent souvent du code d'anciennes versions (`NgModule`, `*ngIf`, `@Input()`), et une réponse plausible n'est pas une réponse vérifiée. Confrontez toujours à la documentation, puis à la compilation, au lint et aux tests.

---

## Installer les outils

Installez ces outils **avant** de commencer. Un seul éditeur suffit : VS Code ou WebStorm.

### Node.js et Git

| Outil | Où le trouver | Installation rapide |
|---|---|---|
| Node.js (version LTS, 24.15 ou plus) | https://nodejs.org | Windows : `winget install OpenJS.NodeJS.LTS` · macOS et Linux : l'installateur du site, ou `nvm install 24` avec [nvm](https://github.com/nvm-sh/nvm) |
| Git | https://git-scm.com | Windows : `winget install Git.Git` · macOS : `brew install git` · Linux : `sudo apt install git` |

Vérifiez dans un nouveau terminal : `node -v` et `git --version`.

### Éditeur, option 1 : Visual Studio Code (gratuit)

**Installer VS Code.** Téléchargez-le sur https://code.visualstudio.com, ou en ligne de commande :

| Système | Commande |
|---|---|
| Windows | `winget install Microsoft.VisualStudioCode` |
| macOS | `brew install --cask visual-studio-code` |
| Linux (Debian, Ubuntu) | le paquet `.deb` du site, ou `sudo snap install code --classic` |

**Installer les extensions.** Elles se trouvent dans la vue *Extensions* (icône des quatre carrés dans la barre de gauche, ou Ctrl+Maj+X / Cmd+Maj+X) : tapez le nom, puis cliquez sur *Install*. Leur page est aussi consultable sur https://marketplace.visualstudio.com.

| Extension | Identifiant | Rôle |
|---|---|---|
| Angular Language Service | `angular.ng-template` | Autocomplétion, erreurs et navigation dans les templates. Maintenue par l'équipe Angular. |
| ESLint | `dbaeumer.vscode-eslint` | Affiche les erreurs de lint directement dans l'éditeur. |
| Prettier – Code formatter | `esbenp.prettier-vscode` | Formate le code selon `.prettierrc`. |
| EditorConfig for VS Code | `editorconfig.editorconfig` | Applique `.editorconfig` (indentation, fin de ligne). |

Méthode la plus rapide : le projet Pokedev déclare ces extensions dans `.vscode/extensions.json` (étape 8 ci-dessous). À l'ouverture du dossier, VS Code propose de les installer : cliquez sur *Install*. Si la proposition n'apparaît pas : palette de commandes (Ctrl+Maj+P / Cmd+Maj+P), puis *Extensions: Show Recommended Extensions*.

En ligne de commande, si la commande `code` est disponible (sur macOS : palette de commandes, *Shell Command: Install 'code' command in PATH*) :

```bash
code --install-extension angular.ng-template
code --install-extension dbaeumer.vscode-eslint
code --install-extension esbenp.prettier-vscode
code --install-extension editorconfig.editorconfig
```

Enfin, activez le formatage à l'enregistrement : *Settings* (Ctrl+, / Cmd+,), cherchez « Format On Save » et cochez-le ; cherchez « Default Formatter » et choisissez *Prettier – Code formatter*.

### Éditeur, option 2 : WebStorm (JetBrains)

WebStorm est l'IDE JavaScript et TypeScript de JetBrains. Il est **gratuit pour un usage non commercial** (apprentissage, projets personnels, open source) ; les étudiants et enseignants peuvent aussi obtenir une licence éducative gratuite sur https://www.jetbrains.com/community/education/. Le support d'Angular est intégré : il n'y a rien à ajouter.

**Installer WebStorm.** Téléchargez-le sur https://www.jetbrains.com/webstorm/download/, ou passez par la **JetBrains Toolbox App** (https://www.jetbrains.com/toolbox-app/), qui installe et met à jour les IDE JetBrains :

| Système | WebStorm | Toolbox App |
|---|---|---|
| Windows | `winget install JetBrains.WebStorm` | `winget install JetBrains.Toolbox` |
| macOS | `brew install --cask webstorm` | `brew install --cask jetbrains-toolbox` |
| Linux | archive `.tar.gz` du site | archive `.tar.gz` du site |

**Activer la licence gratuite.** Au premier lancement : *Help › Register* (ou *Manage Subscriptions* sur l'écran d'accueil), puis *Non-commercial use*, et connectez-vous avec un compte JetBrains.

**Configurer.**
- Ouvrez le dossier du projet avec *File › Open* et choisissez *Trust Project* uniquement pour vos propres projets : un projet non fiable peut exécuter du code via sa configuration. Utilisez une version récente (2026.2 ou plus), qui corrige des failles de ce type.
- Plugin Angular : *Settings › Plugins*, onglet *Installed* : *Angular and AngularJS* doit être activé (il l'est par défaut).
- ESLint : *Settings › Languages & Frameworks › JavaScript › Code Quality Tools › ESLint* : cochez *Automatic ESLint configuration*.
- Prettier : *Settings › Languages & Frameworks › JavaScript › Prettier* : choisissez *Automatic Prettier configuration* et cochez *Run on save*.
- Les scripts de `package.json` (`start`, `test`…) se lancent depuis la marge du fichier (triangle vert) ou la fenêtre *npm*.

### Navigateur : Chrome ou Firefox, avec Angular DevTools

| Navigateur | Où le trouver | Installation rapide |
|---|---|---|
| Google Chrome (ou Edge, Brave : même extension) | https://www.google.com/chrome/ | Windows : `winget install Google.Chrome` · macOS : `brew install --cask google-chrome` |
| Firefox | https://www.mozilla.org/firefox/ | Windows : `winget install Mozilla.Firefox` · macOS : `brew install --cask firefox` · Linux : souvent préinstallé |

**Angular DevTools**, l'extension officielle de l'équipe Angular :
- Chrome : https://chrome.google.com/webstore/detail/angular-developer-tools/ienfalfjdbdpebioblfackkekamfmbnh, puis *Ajouter à Chrome* ;
- Firefox : https://addons.mozilla.org/firefox/addon/angular-devtools/, puis *Ajouter à Firefox*.

Utilisation : ouvrez l'application, puis les outils de développement (F12, ou Ctrl+Maj+I ; Cmd+Option+I sur macOS) et l'onglet **Angular**. Il contient notamment *Components* (arbre des composants, valeurs des entrées et des signaux) et *Profiler*. L'extension ne fonctionne qu'avec une application en mode développement (`ng serve`), pas avec une build de production. Dans Chrome, l'onglet n'apparaît pas sur la page « Nouvel onglet ».

---

## On fait ensemble

### Étape 1 — Vérifier l'environnement

Dans un terminal :

```bash
node -v   # v22.22.3 ou plus, ou v24.15 ou plus
npm -v
git --version
```

La CLI Angular 22 refuse de démarrer avec une version de Node antérieure.

### Étape 2 — Créer le projet

```bash
npx @angular/cli@22 new pokedev --style=css --ssr=false
cd pokedev
npx ng version
```

- `npx` utilise la CLI sans l'installer globalement.
- La CLI pose des questions (par exemple sur les assistants IA) : la réponse par défaut convient.
- `ng version` doit afficher Angular 22.x, TypeScript 6.0.x et Vitest 4.x.

Ouvrez le dossier `pokedev` dans votre éditeur : dans VS Code, *File › Open Folder* ; dans WebStorm, *File › Open*.

### Étape 3 — Découvrir le projet généré

Parcourez l'arborescence dans l'éditeur.

| Fichier | Rôle |
|---|---|
| `package.json` | Dépendances et scripts npm |
| `angular.json` | Configuration du workspace : build, serve, test, lint ; configurations `production` et `development` |
| `tsconfig.json`, `tsconfig.app.json`, `tsconfig.spec.json` | Configuration TypeScript : commune, application, tests |
| `src/main.ts` | Point d'entrée : `bootstrapApplication(App, appConfig)` |
| `src/index.html` | La page HTML qui contient `<app-root>` |
| `src/app/app.config.ts` | Les *providers* globaux : routeur, HTTP, locale… |
| `src/app/app.ts`, `app.html`, `app.css` | Le composant racine |
| `src/app/app.routes.ts` | La table des routes |
| `src/styles.css` | Styles globaux |
| `public/` | Fichiers statiques copiés tels quels : favicon, données JSON… |
| `.prettierrc`, `.editorconfig` | Formatage du code |
| `.vscode/` | Extensions recommandées, configuration de débogage |

Les conventions ont changé récemment. Beaucoup de tutoriels montrent encore les anciennes. Dans Angular 22 :
- les fichiers n'ont plus de suffixe de type : `app.ts`, et non `app.component.ts` ; les générateurs produisent `dex-number-pipe.ts`, `loading-interceptor.ts`, `dev-id-guard.ts` ;
- les classes non plus : `App`, et non `AppComponent` ;
- tous les composants sont *standalone* : il n'y a plus de `NgModule` ;
- les services utilisent `@Service()` ;
- pas de zone.js ;
- les tests utilisent Vitest, et non plus Karma et Jasmine.

### Étape 4 — Lancer l'application

```bash
npx ng serve
```

Ouvrez http://localhost:4200 : la page d'accueil Angular s'affiche. Laissez `ng serve` tourner dans ce terminal et ouvrez-en un second pour les commandes suivantes.

### Étape 5 — Préparer le projet

1. Remplacez **tout** le contenu de `src/app/app.html` (la page de démonstration) par :

`src/app/app.html`

```html
<h1>Pokedev</h1>
```

2. Remplacez `src/app/app.ts` : le composant n'a plus besoin du routeur ni du signal `title`.

`src/app/app.ts`

```ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-root',
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {}
```

3. Dans `src/index.html`, passez la langue en français :

`src/index.html`

```html
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>Pokedev</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

4. Remplacez `src/styles.css` par les styles de base du projet (couleurs, thème sombre automatique, focus visible) :

`src/styles.css`

```css
:root {
  color-scheme: light dark;
  --bg: #ffffff;
  --fg: #1c1c1e;
  --muted: #5f6368;
  --border: #d0d4da;
  --accent: #0b6bcb;
  --on-accent: #ffffff;
  --good-bg: #e3f5e5;
  --error: #b3261e;
  font-family: system-ui, -apple-system, 'Segoe UI', Roboto, sans-serif;
  line-height: 1.5;
}
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #121212;
    --fg: #ececec;
    --muted: #a0a4aa;
    --border: #3a3d42;
    --accent: #6fb1ff;
    --on-accent: #0b1a2a;
    --good-bg: #1d3a22;
    --error: #ff8a80;
  }
}
body { margin: 0; background: var(--bg); color: var(--fg); }
header, main, footer { max-width: 60rem; margin-inline: auto; padding: 1rem; }
.brand { font-weight: 700; font-size: 1.25rem; text-decoration: none; color: inherit; }
a { color: var(--accent); }
input, select, textarea, button { font: inherit; padding: 0.4rem 0.6rem; border: 1px solid var(--border); border-radius: 0.4rem; background: var(--bg); color: var(--fg); }
button { cursor: pointer; }
button:disabled { opacity: 0.5; cursor: not-allowed; }
:focus-visible { outline: 3px solid var(--accent); outline-offset: 2px; }
.error { color: var(--error); }
.muted { color: var(--muted); }
.visually-hidden { position: absolute; width: 1px; height: 1px; overflow: hidden; clip-path: inset(50%); white-space: nowrap; }
footer { color: var(--muted); font-size: 0.875rem; }
```

Résultat : la page affiche « Pokedev » en gros titre, sans rechargement manuel.

### Étape 6 — Les tests

```bash
npx ng test
```

Vitest démarre en mode surveillance : un test échoue, car `src/app/app.spec.ts` cherche encore le texte de la page de démonstration. Corrigez-le :

`src/app/app.spec.ts`

```ts
import { TestBed } from '@angular/core/testing';
import { App } from './app';

describe('App', () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [App],
    })
      .compileComponents();
  });

  it('should create the app', () => {
    const fixture = TestBed.createComponent(App);
    const app = fixture.componentInstance;
    expect(app).toBeTruthy();
  });

  it('should render title', async () => {
    const fixture = TestBed.createComponent(App);
    await fixture.whenStable();
    const compiled = fixture.nativeElement as HTMLElement;
    expect(compiled.textContent).toContain('Pokedev');
  });
});
```

Résultat : dès l'enregistrement, Vitest relance les tests et affiche `Tests 2 passed (2)`. Arrêtez-le avec Ctrl+C.

### Étape 7 — Le lint

```bash
npx ng add angular-eslint
npx ng lint
```

Résultat : `All files pass linting.` La configuration inclut des règles d'accessibilité des templates. Pour voir une erreur, ajoutez `<img src="x.png">` dans `app.html` : le lint signale l'absence d'attribut `alt`. Retirez-la ensuite.

### Étape 8 — Le tour des autres outils

1. **Générateurs.** L'option `--dry-run` affiche les fichiers qui seraient créés, sans rien écrire. Essayez :
   ```bash
   npx ng g component features/dex/dev-card --flat --dry-run
   npx ng g service core/team/team --dry-run
   npx ng g pipe shared/pipes/dex-number --dry-run
   npx ng g directive shared/directives/type-color --dry-run
   npx ng g interceptor core/http/loading --dry-run
   npx ng g guard core/navigation/dev-id --dry-run
   ```
   Sans `--flat`, un composant est créé dans son propre sous-dossier (`features/dex/dev-card/dev-card.ts`). Dans ce cours, les fichiers sont placés directement dans le dossier de la fonctionnalité, d'où `--flat`. Chaque générateur crée aussi un fichier `.spec.ts` ; l'option `--skip-tests` l'évite.
2. **Formatage.** `npx prettier --write src` reformate le code selon `.prettierrc`. Dans VS Code, activez « Format on Save ».
3. **Éditeur.** Dans `app.html`, ajoutez `<p>{{ title }}</p>` : l'erreur « Property 'title' does not exist on type 'App' » est soulignée, dans VS Code (Angular Language Service) comme dans WebStorm. Supprimez la ligne.
   Pour installer les extensions recommandées dans VS Code, remplacez `.vscode/extensions.json` :

`.vscode/extensions.json`

```json
{
  // For more information, visit: https://go.microsoft.com/fwlink/?linkid=827846
  "recommendations": [
    "angular.ng-template",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "editorconfig.editorconfig"
  ]
}
```

   Fermez puis rouvrez le dossier : VS Code propose d'installer les extensions manquantes.
4. **Navigateur.** Sur http://localhost:4200, ouvrez les outils de développement (F12), onglet **Angular**, vue *Components* : l'arbre ne contient pour l'instant que `App`.
5. **Débogage.**
   - VS Code : arrêtez `ng serve` (Ctrl+C), posez un point d'arrêt dans un fichier `.ts` (clic dans la marge), puis appuyez sur F5 et choisissez la configuration **ng serve** de `.vscode/launch.json`. VS Code lance lui-même `npm start`, puis ouvre Chrome ; l'exécution s'arrête sur le point d'arrêt. La configuration **ng test** du même fichier vise l'ancien lanceur Karma et ne fonctionne pas avec Vitest.
   - WebStorm : lancez `ng serve`, puis *Run › Edit Configurations › + › JavaScript Debug*, URL `http://localhost:4200`, et démarrez-la avec l'icône de débogage.
6. **Build.** `npx ng build` produit `dist/pokedev/browser` et affiche la taille des fichiers.
7. **Mises à jour.** `npx ng update` liste les mises à jour disponibles. `npx ng update @angular/core @angular/cli` met à jour **et applique les migrations de code**. Guide : https://angular.dev/update-guide.
8. **Assistant IA.** `npx ng mcp` démarre un serveur MCP qui donne à un assistant de code l'accès à la documentation de la version installée.

### Étape 9 — Versionner

```bash
git add .
git commit -m "chore: projet Pokedev initial"
git log --oneline
```

Si Git est installé et configuré, `ng new` a déjà initialisé le dépôt et fait un premier commit ; `git log` affiche alors les deux commits. Sinon, lancez `git init` avant `git add`.

---

## Vous faites

**Exercice 1 — Reconnaître ce que produit un générateur.** Sans lancer les commandes, dites quels fichiers créent :
1. `npx ng g component features/team/team-page`
2. `npx ng g component features/team/team-page --flat --inline-template --skip-tests`
3. `npx ng g pipe shared/pipes/dex-number`
4. `npx ng g guard core/navigation/dev-id`

Vérifiez ensuite avec `--dry-run`.

<details>
<summary>Correction</summary>

1. `features/team/team-page/team-page.ts`, `.html`, `.css` et `.spec.ts` : un sous-dossier est créé.
2. `features/team/team-page.ts` seul : `--flat` supprime le sous-dossier, `--inline-template` supprime le `.html`, `--skip-tests` supprime le `.spec.ts`. Le `.css` reste.
3. `shared/pipes/dex-number-pipe.ts` et `dex-number-pipe.spec.ts`, classe `DexNumberPipe`.
4. `core/navigation/dev-id-guard.ts` et son test ; la CLI demande le type de garde (`CanActivate` par défaut).

À retenir : les noms de fichiers n'ont plus de point avant le type (`dex-number-pipe.ts`), et les noms de classes gardent le suffixe (`DexNumberPipe`).
</details>

**Exercice 2 — Lire `angular.json`.** Ouvrez `angular.json` et répondez :
1. Où atterrit le résultat de `ng build` ?
2. Quelles sont les deux configurations de build, et laquelle est utilisée par défaut par `ng build` et par `ng serve` ?
3. À partir de quelle taille de code initial la build affiche-t-elle un avertissement, puis une erreur ?

<details>
<summary>Correction</summary>

1. `outputPath` : `dist/pokedev`, et les fichiers du navigateur dans `dist/pokedev/browser`.
2. `production` et `development`. `ng build` utilise `production` (`defaultConfiguration`), `ng serve` utilise `development`.
3. Dans `budgets` : avertissement à 500 kB, erreur à 1 MB pour le budget `initial`.
</details>

**Exercice 3 — Interroger la bonne source.** Pour chacune de ces questions, indiquez où vous chercheriez en premier :
1. « Quels arguments accepte `ng build` ? »
2. « Comment migrer de la version 21 à la 22 ? »
3. « Comment nommer un fichier de service ? »
4. « Pourquoi mon template affiche-t-il une erreur NG8113 ? »

<details>
<summary>Correction</summary>

1. `npx ng build --help`, ou la référence CLI sur angular.dev.
2. https://angular.dev/update-guide, qui liste les étapes et les migrations automatiques selon les versions de départ et d'arrivée.
3. Le guide de style : https://angular.dev/style-guide.
4. Le message complet de la compilation donne le fichier et la ligne ; la référence des erreurs sur angular.dev explique le code NG8113 (un import déclaré mais inutilisé dans le template).
</details>

---

## Check-list

- [ ] Node.js, Git, mon éditeur et ses extensions, mon navigateur et Angular DevTools sont installés.
- [ ] Je sais à quoi sert chaque commande de la CLI et comment prévoir ce qu'un générateur produit.
- [ ] Je sais où chercher de l'aide et pourquoi une réponse d'assistant IA doit être vérifiée.
- [ ] `ng serve`, `ng test` et `ng lint` fonctionnent sur le projet.
- [ ] Le projet est versionné.

---

[← 02 · TypeScript](02-typescript.md) · [Sommaire](README.md) · [04 · Conception →](04-conception.md)
