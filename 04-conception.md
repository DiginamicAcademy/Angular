[← 03 · Les outils et la création du projet](03-environnement-tooling.md) · [Sommaire](README.md)

---

# Phase 04 — Conception : découpage et user stories

## Objectifs

- Découper une maquette en arbre de composants.
- Décider ce que chaque composant reçoit et ce qu'il émet.
- Rédiger des user stories et leurs critères d'acceptation en Gherkin.
- Relier ces critères aux tests qui les vérifieront.

Cette phase ne produit pas de code : elle produit les décisions qui guideront les phases 05 à 12.

---

## On théorise

### 1. Découper une maquette

On part toujours de l'écran, jamais du code. La méthode tient en quatre questions :

1. **Qu'est-ce qui se répète ?** Un élément répété devient un composant, alimenté par une entrée.
2. **Qu'est-ce qui revient sur plusieurs écrans ?** Il va dans `shared/`.
3. **Qui détient la donnée ?** La donnée appartient au composant le plus haut qui en a besoin ; elle descend par les entrées.
4. **Qui décide ?** Un composant d'affichage ne modifie pas l'état global : il **émet** un événement, et le composant page décide.

Cela donne deux catégories :

| Catégorie | Rôle | Exemple |
|---|---|---|
| **Page** (dans `features/`) | Récupère les données, orchestre, décide | `DexPage`, `TeamPage` |
| **Affichage** (dans `features/` ou `shared/`) | Reçoit des entrées, émet des sorties, ne sait pas d'où viennent les données | `DevCard`, `StatBar`, `TypeBadge` |

Un composant d'affichage doit rester utilisable ailleurs, y compris dans un test, sans base de données ni réseau.

### 2. La user story

Le format habituel : **En tant que** rôle, **je veux** action, **afin de** bénéfice.

Une user story n'est pas une spécification technique : elle dit qui, quoi et pourquoi, pas comment. Elle est accompagnée de **critères d'acceptation** : les conditions vérifiables qui permettent de dire « c'est fait ».

### 3. Gherkin

Gherkin est un langage structuré, lisible par une personne non technique, pour écrire ces critères. Les mots-clés en français :

```gherkin
# language: fr
Fonctionnalité: Titre de la fonctionnalité
  En tant que …
  Je veux …
  Afin de …

  Contexte:
    Étant donné que <situation commune à tous les scénarios>

  Scénario: Un cas précis
    Étant donné que <état de départ>
    Quand <action de l'utilisateur>
    Alors <résultat observable>
    Et <autre résultat observable>

  Plan du Scénario: Un cas répété avec des valeurs
    Quand je saisis <entrée>
    Alors je vois <résultat>

    Exemples:
      | entrée | résultat |
      | kube   | 1 dev    |
```

Trois règles :
- **Étant donné** décrit un état, pas une action ; **Quand** décrit une seule action ; **Alors** décrit ce qui est **observable** par l'utilisateur.
- On écrit ce que voit l'utilisateur, jamais l'implémentation. « Alors le signal `teamIds` contient 1 » n'est pas un critère d'acceptation ; « Alors l'équipe compte 1 dev » en est un.
- Un scénario doit être vérifiable sans ambiguïté : il devient directement un cas du plan de tests (phase 11).

Gherkin est aussi le langage de Cucumber, utilisé pour l'automatisation des tests d'acceptation. Ici, on s'en sert comme d'un format d'écriture partagé, sans outil.

---

## On fait ensemble

Voici les écrans de Pokedev, à découper.

```
┌──────────────────────────────────────────────────────────────┐
│ Pokedev              Pokédex   Mon équipe (2)   Créer un dev │
├──────────────────────────────────────────────────────────────┤
│ Pokédex                                                      │
│ Rechercher un dev [ Nom, poste ou langage……… ]               │
│ (Tous) (Front-end) (Back-end) (DevOps) (Data) (Mobile) (Séc.)│
│ 21 dev(s)                                                    │
│ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│ │ (S) #001     │ │ (J) #002     │ │ (S) #003     │           │
│ │ Stagiairon   │ │ Juniorax     │ │ Seniorgon    │           │
│ │ [Front-end]  │ │ [Front-end]  │ │ [Front-end]  │           │
│ │ Total : 180  │ │ Total : 275  │ │ Total : 415  │           │
│ │ [Ajouter]    │ │ [Retirer]    │ │ [Ajouter]    │           │
│ └──────────────┘ └──────────────┘ └──────────────┘           │
└──────────────────────────────────────────────────────────────┘

Fiche d'un dev : avatar, numéro, nom, poste, types, phrase fétiche,
6 barres de statistiques, langages, évolution, bouton d'équipe,
devs du même type.

Mon équipe : membres (avatar, nom, bouton Retirer), types couverts,
moyenne des statistiques.

Créer un dev : formulaire (nom, poste, types, 6 statistiques,
langages, phrase fétiche).
```

### Étape 1 — L'arbre des composants

En appliquant les quatre questions, on obtient :

```
App                         en-tête, navigation, <router-outlet>
├── DexPage                 entrée : type (paramètre d'URL)
│   ├── DevCard × n         entrées : dev, inTeam, teamFull · sortie : teamToggled(id)
│   └── EmptyState          contenu projeté
├── DevDetailPage           entrée : id (paramètre d'URL)
│   ├── DevAvatar           entrées : name, type
│   ├── TypeBadge × 1-2     entrée : type
│   └── StatBar × 6         entrées : label, value, highlight
├── TeamPage
│   ├── DevAvatar × n
│   └── StatBar × 6
├── CreateDevPage
└── NotFoundPage
```

`DevAvatar`, `TypeBadge`, `StatBar` et `EmptyState` reviennent sur plusieurs écrans : ils iront dans `shared/`.

Deux décisions à commenter :
- la carte ne connaît pas l'équipe. Elle reçoit `inTeam` et `teamFull`, et émet `teamToggled`. C'est la page qui modifie l'équipe ;
- le filtre par type est un paramètre d'URL, pas un état interne à la page : le lien devient partageable.

### Étape 2 — La première user story en Gherkin

```gherkin
# language: fr
Fonctionnalité: Parcourir le pokédex
  En tant que visiteur
  Je veux parcourir et filtrer la liste des devs
  Afin de trouver rapidement celui qui m'intéresse

  Contexte:
    Étant donné que le pokédex contient 21 devs

  Scénario: Voir tous les devs
    Quand j'ouvre la page du pokédex
    Alors je vois 21 cartes de dev
    Et chaque carte affiche un numéro, un nom, un poste et un ou deux types

  Scénario: Filtrer par type
    Quand je choisis le filtre "Data"
    Alors je ne vois que les devs de type Data
    Et l'adresse de la page contient le filtre choisi

  Plan du Scénario: Rechercher par texte
    Quand je saisis <recherche> dans le champ de recherche
    Alors je vois <nombre> dev(s)

    Exemples:
      | recherche | nombre |
      | kube      | 1      |
      | python    | 4      |
      | zzz       | 0      |

  Scénario: Aucun résultat
    Quand je saisis "zzz" dans le champ de recherche
    Alors je vois le message "Aucun dev ne correspond à la recherche."
    Et je peux effacer la recherche d'un clic
```

Remarquez ce qui n'y figure pas : ni `signal`, ni `httpResource`, ni nom de composant. Ces scénarios resteraient valables avec une autre technologie.

---

## Vous faites

**Exercice 1 — Compléter l'arbre.** Pour la fiche d'un dev et pour la page « Mon équipe », listez les composants, et pour chacun : ses entrées, ses sorties, et s'il appartient à `features/` ou à `shared/`. Indiquez aussi quel composant détient l'état de l'équipe.

<details>
<summary>Correction</summary>

| Composant | Dossier | Entrées | Sorties |
|---|---|---|---|
| `DevDetailPage` | `features/detail` | `id` (paramètre de route) | — |
| `DevAvatar` | `shared/ui` | `name`, `type` | — |
| `TypeBadge` | `shared/ui` | `type` | — |
| `StatBar` | `shared/ui` | `label`, `value`, `max`, `highlight` | — |
| `EmptyState` | `shared/ui` | contenu projeté | — |
| `TeamPage` | `features/team` | — | — |

L'état de l'équipe n'est détenu par aucun composant : il est partagé par trois écrans (en-tête, pokédex, fiche) et doit survivre à un rechargement. Il ira donc dans un **service**, `Team` (phase 06), rendu persistant en phase 09. C'est la réponse à la question « qui détient la donnée ? » quand la réponse est « plusieurs écrans » : ni l'un ni l'autre, un service.
</details>

**Exercice 2 — Écrire les user stories restantes.** Rédigez, au format « En tant que…, je veux…, afin de… », les user stories correspondant aux trois autres écrans, puis à la persistance. Visez cinq stories.

<details>
<summary>Correction</summary>

1. En tant que visiteur, je veux consulter la fiche d'un dev, afin de voir ses statistiques, ses langages et ses évolutions.
2. En tant que joueur, je veux composer une équipe de six devs au plus, afin de couvrir un maximum de types.
3. En tant que joueur, je veux voir la couverture et la moyenne de mon équipe, afin de savoir ce qui lui manque.
4. En tant que joueur, je veux retrouver mon équipe en revenant sur le site, afin de ne pas la recomposer.
5. En tant que joueur, je veux créer mon propre dev, afin de l'ajouter au pokédex.
</details>

**Exercice 3 — Traduire deux stories en Gherkin.** Écrivez les scénarios de la story « composer une équipe » et de la story « créer un dev ». Pour chacune, prévoyez au moins un scénario nominal et un scénario d'erreur ou de limite.

<details>
<summary>Correction</summary>

```gherkin
# language: fr
Fonctionnalité: Composer une équipe
  En tant que joueur
  Je veux composer une équipe de six devs au plus
  Afin de couvrir un maximum de types

  Scénario: Ajouter un dev à l'équipe
    Étant donné que mon équipe est vide
    Quand j'ajoute Stagiairon à mon équipe
    Alors mon équipe compte 1 dev
    Et le compteur de l'en-tête affiche 1
    Et la carte de Stagiairon propose de le retirer

  Scénario: Limite de six devs
    Étant donné que mon équipe compte 6 devs
    Quand je consulte le pokédex
    Alors les boutons d'ajout des autres devs sont désactivés
    Mais je peux encore retirer un dev de mon équipe

  Scénario: Équipe conservée
    Étant donné que mon équipe compte 2 devs
    Quand je recharge la page
    Alors mon équipe compte toujours 2 devs

  Scénario: Couverture des types
    Étant donné que mon équipe est composée de Tabulis et Requêtor
    Quand j'ouvre la page de mon équipe
    Alors je vois que 2 types sur 6 sont couverts
    Et je vois la liste des types manquants
```

```gherkin
# language: fr
Fonctionnalité: Créer un dev
  En tant que joueur
  Je veux créer mon propre dev
  Afin de l'ajouter au pokédex

  Scénario: Création réussie
    Étant donné que je suis sur le formulaire de création
    Quand je saisis un nom, un poste, un type et au moins un langage valides
    Et que j'enregistre
    Alors la fiche du dev créé s'affiche
    Et le dev apparaît dans le pokédex, même après rechargement

  Scénario: Nom déjà utilisé
    Quand je saisis un nom qui existe déjà dans le pokédex
    Alors je vois le message "Ce nom est déjà pris."
    Et l'enregistrement reste impossible

  Scénario: Total de statistiques trop élevé
    Quand la somme des six statistiques dépasse 420
    Alors je vois un message qui l'indique
    Et l'enregistrement reste impossible

  Scénario: Quitter sans enregistrer
    Étant donné que j'ai commencé à remplir le formulaire
    Quand je navigue vers une autre page
    Alors une confirmation m'est demandée
    Et je reste sur le formulaire si je refuse
```

Chacun de ces scénarios deviendra un cas du plan de tests de la phase 11, et la plupart seront automatisés.
</details>

---

## Check-list

- [ ] L'arbre des composants est écrit, avec les entrées et les sorties de chacun.
- [ ] Je sais justifier qu'un état partagé va dans un service et non dans un composant.
- [ ] Les user stories couvrent les quatre écrans et la persistance.
- [ ] Chaque story a des critères d'acceptation en Gherkin, observables par l'utilisateur.

---

[← 03 · Les outils et la création du projet](03-environnement-tooling.md) · [Sommaire](README.md) · [05 · Composants et signaux →](05-composants-signaux.md)
