[← Chapitre précédent](03-Slides.md) · [Sommaire](Readme.md)

# Créditer l'auteur et les contributeurs

Un support de cours circule détaché de son dépôt : téléchargé, transféré,
imprimé. Il doit donc porter en lui-même le nom de qui l'a écrit. Le workflow
s'en charge, à partir de deux sources bien distinctes.

| Source | Nature | Où elle apparaît |
|---|---|---|
| Le fichier `AUTHORS` | une décision éditoriale, écrite à la main | page de garde et ouverture des crédits |
| L'historique git | un relevé, calculé | liste des contributeurs, en fin de document |

La distinction est le cœur du sujet : **signer n'est pas contribuer**. Corriger
trois coquilles inscrit au relevé, pas au générique.

## Le fichier `AUTHORS`

Un fichier `AUTHORS` à la racine du dépôt, une personne par ligne :

```text
# Les signataires du cours, dans l'ordre où ils doivent apparaître.
Jean-François Vial
```

Les lignes vides et celles commençant par `#` sont ignorées, ce qui laisse la
place aux commentaires. L'ordre du fichier est respecté : c'est un choix
éditorial, pas un classement.

Les noms sont ensuite assemblés en une phrase française — « A », « A et B »,
« A, B et C » — qui sert à deux endroits :

- sur la **page de garde**, en regard du libellé « Par » ;
- en tête de la section **Crédits**, sous la forme « Cours de A et B. ».

Sans `AUTHORS`, ces deux mentions disparaissent : la page de garde ne porte plus
que le dépôt, la source et la version, et les crédits s'ouvrent sur « Ont
contribué à ce cours ».

## Les contributeurs

La liste est tirée de l'historique git, les plus prolifiques d'abord, puis
enrichie par GitHub. Trois étapes :

1. **L'historique** fournit un nom et une adresse par commit.
2. **GitHub** relie l'adresse à un compte, quand elle en a un. Le nom du compte
   l'emporte alors sur celui du commit — souvent un pseudonyme — et son adresse
   publique sur l'adresse de commit.
3. **Le tri** écarte les robots, les adresses `noreply`, les doublons, et toute
   personne déjà citée dans `AUTHORS`.

L'enrichissement est du « au mieux » : sans réseau ni droits suffisants, le
workflow émet un avertissement et retombe sur les identités des commits. Le
document se construit dans tous les cas.

## Deux réglages qui changent le résultat

**Une même personne, plusieurs adresses.** Un poste au bureau, un autre à la
maison, et voilà deux contributeurs pour une seule personne. Un fichier
`.mailmap` à la racine les réunit :

```text
Jean-François Vial <jf@exemple.fr> <ancienne-adresse@exemple.fr>
```

C'est git lui-même qui l'applique, avant tout le reste.

**Le nom du signataire doit correspondre.** Une personne n'est retirée de la
liste des contributeurs que si le nom calculé est **exactement** celui d'une
ligne d'`AUTHORS`. Or ce nom calculé peut venir du compte GitHub. Si `AUTHORS`
porte « Jean-François Vial » et que le compte affiche « Jeff Vial », la même
personne est créditée deux fois, une fois comme signataire et une fois comme
contributeur. Le remède : aligner le nom du compte GitHub et la ligne
d'`AUTHORS`.

## Ce que le document affiche

La section « Crédits » ferme le support. Elle n'est ajoutée que s'il y a quelque
chose à dire : sans signataire ni contributeur, elle n'existe pas.

```markdown
## Crédits

Cours de Jean-François Vial.

Ont également contribué :

- Une Contributrice <elle@exemple.fr>
- Un Contributeur
```

Une personne sans adresse publique n'apparaît que par son nom : une adresse
`noreply` n'apprendrait rien à personne.

## Les robots ne sont pas crédités

Les commits signés par un assistant — Claude, un `[bot]` quelconque — sont
écartés de la liste. Les lignes `Co-Authored-By` ne sont de toute façon pas
lues : seul l'auteur du commit compte.

---

[← Chapitre précédent](03-Slides.md) · [Sommaire](Readme.md)
