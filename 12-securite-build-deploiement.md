[← 11 · Tests](11-tests.md) · [Sommaire](README.md)

---

# Phase 11 — Sécurité, build et déploiement

## Objectifs

- Connaître les protections d'Angular contre le XSS et leurs limites.
- Mettre en place une Content Security Policy en production.
- Comprendre ce que produit une build de production.
- Mettre en place une intégration et un déploiement continus vers GitHub Pages.
- Lire du code Angular écrit avant les signaux et connaître les migrations automatiques.

Point d'arrivée : Pokedev en ligne, publié automatiquement à chaque push si le lint, les tests et la build réussissent.

---

## On théorise

### 1. La sécurité côté front

| Menace ou risque | Protection |
|---|---|
| **XSS** : injection de script via des données | Angular **échappe** toute interpolation `{{ }}` et **assainit** `[innerHTML]`. On ne contourne jamais ces protections avec `bypassSecurityTrust…` pour des données saisies ou reçues. |
| Script injecté malgré tout | **Content Security Policy** : le navigateur refuse les scripts non autorisés et les requêtes vers des domaines non listés. |
| Données externes malformées | Validation à la frontière : `parseDevs`, lecture défensive du `localStorage`. |
| Secrets dans le code front | **Aucun secret dans le front** : tout le JavaScript livré est lisible. Une API à clé exige un backend. |
| Contrôle d'accès | Une garde de route n'est pas une sécurité : le code front est modifiable par l'utilisateur. Les droits se vérifient côté serveur. |
| Dépendances vulnérables | `npm audit`, mises à jour régulières (`ng update`). |
| Accessibilité | Règles d'accessibilité d'`angular-eslint`, audit Lighthouse. |

### 2. La build de production

`npx ng build` produit, dans `dist/pokedev/browser/` :
- du JavaScript **compilé à l'avance** (AOT), **minifié** et débarrassé du code inutilisé (*tree-shaking*) ;
- des **noms de fichiers hachés** (`main-EOIDTTWO.js`) : un fichier modifié change de nom, le navigateur peut donc garder les autres en cache longtemps ;
- **un fichier par page** chargée à la demande (`dex-page`, `dev-detail-page`, `team-page`…) ;
- une vérification des **budgets** définis dans `angular.json` : avertissement au-delà de 500 kB, erreur au-delà de 1 MB pour le code initial.

Mesure sur Pokedev : environ 86 kB transférés au chargement initial (305 kB avant compression) ; chaque page ajoute quelques kilo-octets.

### 3. Héberger une SPA

Le site est **statique**. Une URL profonde comme `/devs/11` n'existe pas en tant que fichier : le serveur doit renvoyer `index.html`, puis le routeur Angular prend le relais.

| Hébergeur | Solution |
|---|---|
| GitHub Pages | Copier `index.html` en `404.html`, et définir `--base-href` au nom du dépôt |
| Netlify | Fichier `_redirects` : `/* /index.html 200` |
| Nginx | `try_files $uri $uri/ /index.html;` |

### 4. CI/CD

```
git push ──▶ GitHub Actions ──▶ npm ci ──▶ lint ──▶ tests ──▶ build ──▶ (main seulement) déploiement
                                  └──── un échec arrête tout : rien de cassé n'est publié ────┘
```

- **Intégration continue (CI)** : chaque modification est vérifiée automatiquement.
- **Déploiement continu (CD)** : chaque modification validée sur `main` est publiée.
- **`npm ci`** installe exactement les versions du `package-lock.json` : l'installation est reproductible.

---

## On fait ensemble

Point de départ : le projet à la fin de la phase 11.

### Étape 1 — Démonstration XSS

Créez `src/app/xss-demo.ts` :

`src/app/xss-demo.ts`

```ts
import { Component, signal } from '@angular/core';

/** Démonstration : ce qu'Angular fait d'un contenu malveillant saisi par un utilisateur. */
@Component({
  selector: 'app-xss-demo',
  template: `
    <label for="phrase">Phrase fétiche</label>
    <input id="phrase" [value]="phrase()" (input)="phrase.set(input.value)" #input />

    <h3>Interpolation (texte)</h3>
    <p>{{ phrase() }}</p>

    <h3>[innerHTML] (HTML assaini)</h3>
    <p [innerHTML]="phrase()"></p>
  `,
})
export class XssDemo {
  protected readonly phrase = signal('<img src="x" onerror="alert(\'piraté\')"><b>Salut</b>');
}
```

Affichez-le temporairement : dans `app.ts`, ajoutez `XssDemo` aux `imports` ; dans `app.html`, ajoutez `<app-xss-demo />` juste avant `<router-outlet />`.

Résultat attendu :
- sous « Interpolation (texte) », la balise `<img src="x" onerror=…>` est affichée **en texte** ;
- sous « [innerHTML] (HTML assaini) », « **Salut** » apparaît en gras, mais aucune alerte ne s'ouvre : l'attribut `onerror` a été supprimé. La console affiche l'avertissement « sanitizing HTML stripped some content » ;
- modifiez le champ : les deux zones se mettent à jour, toujours sans exécuter de script.

Retirez ensuite `<app-xss-demo />` et l'import de `app.ts`. Le fichier peut rester dans le projet comme référence.

### Étape 2 — La Content Security Policy

La CSP ne s'applique qu'en production : en développement, elle bloquerait le rechargement automatique.

1. Créez `src/index.prod.html`, copie de `index.html` avec la balise CSP :

`src/index.prod.html`

```html
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>Pokedev</title>
  <base href="/">
  <meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; connect-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

2. Dans `angular.json`, dans `projects › pokedev › architect › build › configurations › production`, ajoutez les propriétés `index` et `optimization`. Le fichier complet :

`angular.json`

```json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "cli": {
    "packageManager": "npm",
    "schematicCollections": [
      "angular-eslint"
    ]
  },
  "newProjectRoot": "projects",
  "projects": {
    "pokedev": {
      "projectType": "application",
      "schematics": {},
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular/build:application",
          "options": {
            "browser": "src/main.ts",
            "tsConfig": "tsconfig.app.json",
            "assets": [
              {
                "glob": "**/*",
                "input": "public"
              }
            ],
            "styles": [
              "src/styles.css"
            ]
          },
          "configurations": {
            "production": {
              "budgets": [
                {
                  "type": "initial",
                  "maximumWarning": "500kB",
                  "maximumError": "1MB"
                },
                {
                  "type": "anyComponentStyle",
                  "maximumWarning": "4kB",
                  "maximumError": "8kB"
                }
              ],
              "outputHashing": "all",
              "index": {
                "input": "src/index.prod.html",
                "output": "index.html"
              },
              "optimization": {
                "scripts": true,
                "styles": {
                  "minify": true,
                  "inlineCritical": false
                },
                "fonts": true
              }
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true
            }
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          "builder": "@angular/build:dev-server",
          "configurations": {
            "production": {
              "buildTarget": "pokedev:build:production"
            },
            "development": {
              "buildTarget": "pokedev:build:development"
            }
          },
          "defaultConfiguration": "development"
        },
        "test": {
          "builder": "@angular/build:unit-test"
        },
        "lint": {
          "builder": "@angular-eslint/builder:lint",
          "options": {
            "lintFilePatterns": [
              "src/**/*.ts",
              "src/**/*.html"
            ]
          }
        }
      }
    }
  }
}
```

3. Construisez et servez la version de production :

```bash
npx ng build
npx ng serve --configuration production
```

Résultat attendu : sur http://localhost:4200, l'application fonctionne normalement, sans aucune erreur dans la console. Dans l'onglet « Éléments », la balise `<meta http-equiv="Content-Security-Policy">` est présente dans `<head>`.

- **`connect-src 'self'`** : l'application ne peut interroger que son propre domaine, d'où vient `devs.json`.
- **`inlineCritical: false`** : par défaut, la build insère le CSS critique avec un attribut `onload` que la CSP bloquerait.
- **`'unsafe-inline'` pour les styles** : Angular injecte les styles des composants dans des balises `<style>`. On peut s'en passer avec un *nonce*, qui exige un serveur capable d'en générer un par requête ; GitHub Pages ne le fait pas.

### Étape 3 — Le workflow

Créez `.github/workflows/ci.yml` à la racine du projet :

`.github/workflows/ci.yml`

```yaml
name: CI et déploiement

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v6
        with:
          node-version: 24
          cache: npm

      - name: Installer les dépendances
        run: npm ci

      - name: Lint
        run: npx ng lint

      - name: Tests unitaires
        run: npx ng test --watch=false

      - name: Build de production
        # Le site est servi sous https://<compte>.github.io/<dépôt>/
        run: npx ng build --base-href "/${{ github.event.repository.name }}/"

      - name: Page 404 pour les liens profonds de la SPA
        run: cp dist/pokedev/browser/index.html dist/pokedev/browser/404.html

      - name: Préparer l'artefact GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-pages-artifact@v5
        with:
          path: dist/pokedev/browser

  deploy:
    if: github.ref == 'refs/heads/main'
    needs: build
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Déployer sur GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

- `permissions` : le principe du moindre privilège. Seul le job `deploy` peut écrire sur Pages ;
- `github.event.repository.name` : le `base-href` s'adapte au nom du dépôt ;
- versions des actions (septembre 2026) : `checkout@v6`, `setup-node@v6`, `upload-pages-artifact@v5`, `deploy-pages@v4`.

Pour vérifier en local ce que fera la CI :

```bash
npm ci
npx ng lint
npx ng test --watch=false
npx ng build --base-href /pokedev/
```

Ouvrez `dist/pokedev/browser/index.html` : la balise `<base href="/pokedev/">` a remplacé `<base href="/">`.

### Étape 4 — Publier

1. Sur GitHub, créez un dépôt **public** nommé `pokedev`, sans README.
2. Reliez le projet et poussez :
   ```bash
   git add .
   git commit -m "ci: intégration et déploiement continus"
   git branch -M main
   git remote add origin https://github.com/<compte>/pokedev.git
   git push -u origin main
   ```
3. Dans le dépôt : **Settings → Pages → Build and deployment → Source : GitHub Actions**.
4. Onglet **Actions** : le workflow « CI et déploiement » s'exécute. Si le job `deploy` a échoué parce que Pages n'était pas encore activé, cliquez sur « Re-run all jobs ».

Résultat attendu :
- l'application en ligne sur `https://<compte>.github.io/pokedev/` ;
- `https://<compte>.github.io/pokedev/devs/11` ouvre directement la fiche de Requêtor, grâce à la copie de `index.html` en `404.html` ;
- aucune erreur dans la console.

Faites échouer un test (par exemple, remplacez `'#007'` par `'#7'` dans `dev-rules.spec.ts`), commitez et poussez : l'étape « Tests unitaires » passe au rouge et le job `deploy` n'est pas exécuté. Rétablissez le test et poussez à nouveau.

---

## Vous faites

**Exercice 1 — Le README.** Rédigez le `README.md` du projet : présentation, prérequis, commandes, architecture, déploiement.

<details>
<summary>Exemple</summary>

`README.md`

````markdown
# Pokedev

Un pokédex de développeurs, construit avec Angular 22. Tous les devs sont fictifs.

## Prérequis

- Node.js 22.22.3 ou plus, ou 24.15 ou plus (exigence d'Angular 22)
- npm

## Commandes

| Commande | Rôle |
|---|---|
| `npm install` | Installer les dépendances |
| `npx ng serve` | Serveur de développement sur http://localhost:4200 |
| `npx ng test` | Tests unitaires (Vitest) en mode surveillance |
| `npx ng test --watch=false` | Tests unitaires, exécution unique |
| `npx ng lint` | Analyse statique (angular-eslint) |
| `npx ng build` | Build de production dans `dist/pokedev/browser` |

## Architecture

```
public/data/devs.json   les données du pokédex
src/app/
├── domain/        modèle et règles métier : aucune dépendance Angular
├── core/          services transverses
│   ├── data/          chargement, validation et ajout des devs
│   ├── http/          intercepteur et indicateur de chargement
│   ├── navigation/    gardes et resolver de routes
│   ├── storage/       signal persisté dans le localStorage
│   └── team/          l'équipe de l'utilisateur
├── features/      une page par dossier : dex, detail, team, create, not-found
├── shared/        composants d'interface, directive et pipes réutilisables
└── testing/       données de test
```

## Déploiement

Le workflow `.github/workflows/ci.yml` exécute lint, tests et build à chaque push,
puis publie sur GitHub Pages depuis `main`.
Activer au préalable : Settings → Pages → Source : **GitHub Actions**.
````
</details>

**Exercice 2 — Audit.**
1. Lancez `npm audit` et notez les vulnérabilités éventuelles et leur gravité.
2. Lancez un audit **Lighthouse** (outils de développement de Chromium) sur la version en ligne : Performance, Accessibilité, Bonnes pratiques. Relevez les scores et corrigez un point signalé.
3. Dans la version en ligne, ouvrez la console : aucune erreur de CSP ne doit apparaître.

**Exercice 3 — Lire du code historique.** En entreprise, vous rencontrerez du code écrit avant les signaux. Réécrivez ce composant en Angular moderne :

```ts
// Code historique (Angular 16 et avant) : à savoir lire, à ne plus écrire
@NgModule({
  declarations: [AppComponent, DevCardComponent],
  imports: [BrowserModule, HttpClientModule],
  bootstrap: [AppComponent],
})
export class AppModule {}

@Injectable({ providedIn: 'root' })
export class DevService {
  constructor(private http: HttpClient) {}
  getDevs(): Observable<Dev[]> {
    return this.http.get<Dev[]>('data/devs.json');
  }
}

@Component({ selector: 'app-dev-list', templateUrl: './dev-list.component.html' })
export class DevListComponent implements OnInit {
  @Input() type!: string;
  @Output() selected = new EventEmitter<number>();
  devs$!: Observable<Dev[]>;

  constructor(private devService: DevService) {}

  ngOnInit(): void {
    this.devs$ = this.devService.getDevs();
  }
}
```

```html
<ul *ngIf="devs$ | async as devs; else loading">
  <li *ngFor="let dev of devs" (click)="selected.emit(dev.id)">{{ dev.name }}</li>
</ul>
<ng-template #loading>Chargement…</ng-template>
```

<details>
<summary>Correction</summary>

| Historique | Moderne |
|---|---|
| `NgModule`, `declarations` | Composants standalone, `imports` dans le composant |
| `app.component.ts`, `AppComponent` | `app.ts`, `App` |
| `@Injectable({ providedIn: 'root' })` | `@Service()` |
| Injection par le constructeur | `inject()` |
| `@Input()`, `@Output()` + `EventEmitter` | `input()`, `output()` |
| `*ngIf`, `*ngFor`, `[ngSwitch]` | `@if`, `@for` (avec `track`), `@switch` |
| `Observable` + pipe `async` | Signaux, `httpResource` |
| `ngOnInit` pour charger des données | Initialisation de champ, `computed`, ressources |
| zone.js | Zoneless, piloté par les signaux |
| Karma, Jasmine | Vitest |

Le composant réécrit correspond à `DexPage` (phase 06) et `DevCard` (phase 07) : `httpResource` dans un service, `input()` et `output()`, `@if` / `@for`. Remarquez aussi le `<li>` cliquable du code historique : il n'est pas accessible au clavier. La version moderne utilise un `<button>`.

Angular fournit des **migrations automatiques**, à lancer sur un projet versionné :

```bash
npx ng generate @angular/core:standalone
npx ng generate @angular/core:control-flow
npx ng generate @angular/core:inject
npx ng generate @angular/core:signal-input-migration
npx ng generate @angular/core:output-migration
npx ng generate @angular/core:service
```
</details>

---

## Check-list

- [ ] Je sais comment Angular protège contre le XSS et comment ne pas contourner cette protection.
- [ ] La build de production contient la CSP et l'application fonctionne sans erreur de console.
- [ ] La CI exécute lint, tests et build ; le déploiement n'a lieu que si tout passe.
- [ ] L'application est en ligne et les URL profondes fonctionnent.
- [ ] Je sais lire du code Angular historique.

---

[← 11 · Tests](11-tests.md) · [Sommaire](README.md) · [13 · Projet Time To →](13-projet-time-to.md)
