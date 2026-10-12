[← Chapitre précédent](12-securite-build-deploiement.md) · [Sommaire](README.md)

---

# Phase 13 — Projet Time To (facultatif)

> **Projet d'entraînement, facultatif et non noté.** Il sert à vous exercer. L'évaluation du cours passe par le [TP noté](https://github.com/DiginamicAcademy/TP-Angular22).

Avec Pokedev, chaque étape était guidée. Ici, vous réalisez seul une application complète, de la création du projet à sa mise en ligne, à partir d'un cahier des charges : c'est la situation d'un vrai projet. Toutes les notions nécessaires ont été vues aux chapitres 02 à 12.

## Objectifs

- Mener seul un projet Angular de bout en bout : conception, développement, tests, mise en ligne.
- Réinvestir les notions du cours sur un sujet nouveau, sans pas-à-pas.
- Passer d'un cahier des charges (user stories, règles métier, API) à du code testé.

Point d'arrivée : Time To en ligne, qui trouve les meilleurs créneaux de la semaine pour une activité d'extérieur.

## Comment aborder le projet

- **Quand.** Après le chapitre 12. Chaque exigence technique (section 5) renvoie au chapitre qui l'enseigne.
- **Jusqu'où.** Commencez par le socle : US1 à US6 (section 2). Les user stories complémentaires, US7 à US10, sont des bonus, à ajouter une fois le socle en ligne. Si le temps manque, un projet partiel reste un bon entraînement : suivez les jalons dans l'ordre (section 6) et arrêtez-vous après l'un d'eux.
- **Les aides.** La [boîte à outils](https://github.com/DiginamicAcademy/Angular22-toolbox) résume chaque notion en une fiche. Le code de Pokedev sert de modèle d'architecture.
- **Le corrigé.** `time-to-corrige.zip`, à la racine du dépôt du cours, contient une réalisation possible. Ouvrez-le après avoir cherché : pour vous débloquer sur un jalon, ou pour comparer vos choix une fois le projet terminé.

---

## 1. Le besoin

Time To répond à une question : **« Quand puis-je faire mon activité en extérieur cette semaine ? »**

L'utilisateur choisit un lieu et une activité (vélo, randonnée, course à pied, étendre le linge). L'application récupère les prévisions météo heure par heure sur 7 jours, attribue à chaque heure un score de 0 à 100 selon l'activité, puis affiche les meilleurs créneaux.

```
lieu + activité
      │
      ▼
prévisions heure par heure, sur 7 jours        API Open-Meteo (section 4)
      │
      ▼
score de chaque heure, de 0 à 100              règles métier (section 3)
      │
      ▼
les 3 meilleurs créneaux                       règles métier (section 3)
```

L'application n'a pas de backend : elle interroge directement l'API publique Open-Meteo depuis le navigateur.

---

## 2. User stories

### Le socle

| Id | User story |
|---|---|
| US1 | En tant qu'utilisateur, je veux chercher un lieu par son nom et choisir parmi les résultats, afin de consulter ses prévisions. |
| US2 | En tant qu'utilisateur, je veux choisir une activité, afin que le score corresponde à ce que je veux faire. |
| US3 | En tant qu'utilisateur, je veux voir les 3 meilleurs créneaux de la semaine (début, fin, score moyen, températures minimale et maximale), afin de planifier ma sortie. |
| US4 | En tant qu'utilisateur, je veux voir le détail heure par heure d'une journée (heure, conditions, température, vent, pluie, score), afin de comprendre le score. |
| US5 | En tant qu'utilisateur, je veux partager un lien vers une prévision (lieu et activité), afin qu'une autre personne voie la même chose. |
| US6 | En tant qu'utilisateur, je veux être informé clairement pendant le chargement, en cas d'erreur réseau ou si aucun créneau n'est trouvé. |

### Pour aller plus loin

| Id | User story |
|---|---|
| US7 | En tant qu'utilisateur, je veux utiliser ma position actuelle, afin de ne rien saisir. |
| US8 | En tant qu'utilisateur, je veux ajuster les seuils d'une activité (températures, vent, pluie), afin que le score corresponde à mes préférences. Les réglages sont conservés après rechargement. |
| US9 | En tant qu'utilisateur sensible à la pollution, je veux que la qualité de l'air soit prise en compte pour la randonnée et la course. |
| US10 | En tant qu'utilisateur, je veux retrouver le dernier lieu consulté en ouvrant l'application. |

Comme au chapitre [04 · Conception](04-conception.md), écrivez d'abord les critères d'acceptation de chaque user story en Gherkin : ils lèvent les ambiguïtés et deviennent les cas de votre plan de tests.

---

## 3. Règles métier

### Profils d'activité par défaut

| Activité | Température idéale | Tolérance | Vent max | Pluie max | Journée seulement | Qualité de l'air max |
|---|---|---|---|---|---|---|
| Vélo | 15 à 25 °C | 10 °C | 30 km/h | 30 % | oui | — |
| Randonnée | 12 à 24 °C | 10 °C | 40 km/h | 30 % | oui | 60 |
| Course à pied | 8 à 20 °C | 10 °C | 35 km/h | 40 % | non | 40 |
| Étendre le linge | 18 à 35 °C | 12 °C | 45 km/h | 10 % | oui | — |

- **Tolérance** : l'écart hors de la plage idéale à partir duquel le score tombe à 0.
- **Journée seulement** : l'activité ne se pratique pas la nuit, c'est-à-dire quand `is_day` vaut 0.
- **Qualité de l'air max** : seuil de l'indice européen de qualité de l'air. Il ne sert que pour US9.

### Score d'une heure

Le score combine quatre facteurs compris entre 0 et 1 : 1 quand la condition est idéale, 0 quand elle rend l'activité impossible.

```
score = arrondi(100 × facteurTempérature × facteurVent × facteurPluie × facteurAir)
```

Exception : si l'activité se pratique en journée seulement et qu'il fait nuit, le score vaut 0.

| Facteur | Règle | Exemples pour le vélo |
|---|---|---|
| Température | 1 dans la plage idéale, bornes comprises. En dehors : `1 − écart / tolérance`, où l'écart est la distance à la borne la plus proche ; 0 dès que l'écart atteint la tolérance. | 20 °C → 1 ; 10 °C (écart 5) → 0,5 ; 5 °C (écart 10) → 0 |
| Vent | 1 jusqu'à la moitié du vent max ; diminue ensuite linéairement jusqu'à 0 au vent max, et reste à 0 au-delà. | 15 km/h → 1 ; 22,5 km/h → 0,5 ; 30 km/h → 0 |
| Pluie | 0,8 si la probabilité est inconnue ; 0 si elle dépasse la pluie max ; sinon `1 − probabilité / 200`. | 20 % → 0,9 ; 30 % → 0,85 ; 40 % → 0 ; inconnue → 0,8 |
| Air | 1 si l'indice est inconnu ou si l'activité n'a pas de seuil ; sinon 1 jusqu'au seuil compris, 0,3 au-dessus. Sans US9, il vaut toujours 1. | Toujours 1 : le vélo n'a pas de seuil |

Exemple complet, pour le vélo : une heure de jour à 20 °C, avec un vent de 22,5 km/h et 20 % de risque de pluie, a pour score `arrondi(100 × 1 × 0,5 × 0,9 × 1)`, soit 45. La même heure de nuit a pour score 0.

### Créneaux

Un créneau est une suite d'**au moins 2 heures consécutives** dont chaque score est **au moins 60**.

- Les heures déjà passées sont ignorées ; l'heure en cours compte encore.
- Le score moyen d'un créneau est la moyenne arrondie des scores de ses heures.
- Les créneaux sont triés par score moyen décroissant, puis par durée décroissante. On affiche les 3 premiers.

Exemple : de 9 h à 14 h, les scores sont 50, 70, 80, 90, 55 et 65. Les heures de 10 h, 11 h et 12 h forment un créneau de 3 heures, de score moyen 80. L'heure de 14 h, seule, ne forme pas de créneau.

---

## 4. L'API Open-Meteo

Trois appels : la recherche de lieu, les prévisions et, pour US9 seulement, la qualité de l'air.

L'API est gratuite pour un usage non commercial, sans clé ni compte, et appelable depuis un navigateur (CORS autorisé). Limite : 10 000 appels par jour. Les données sont sous licence **CC BY 4.0** : la licence exige d'afficher dans l'application la mention « Données météo : Open-Meteo (CC BY 4.0) », avec un lien vers [open-meteo.com](https://open-meteo.com).

Documentation : [open-meteo.com/en/docs](https://open-meteo.com/en/docs)

### Recherche de lieu

```
GET https://geocoding-api.open-meteo.com/v1/search?name=Montpellier&count=8&language=fr
```

```json
{
  "results": [
    { "id": 2992166, "name": "Montpellier", "latitude": 43.61, "longitude": 3.88,
      "country": "France", "admin1": "Occitanie" }
  ]
}
```

Sans résultat, la propriété `results` est **absente**.

### Prévisions

```
GET https://api.open-meteo.com/v1/forecast
    ?latitude=43.61&longitude=3.88
    &current=temperature_2m,weather_code
    &hourly=temperature_2m,precipitation_probability,wind_speed_10m,weather_code,is_day
    &timezone=auto&forecast_days=7
```

```json
{
  "latitude": 43.61, "longitude": 3.88, "timezone": "Europe/Paris",
  "current": { "time": "2026-09-19T09:00", "temperature_2m": 19, "weather_code": 0 },
  "hourly": {
    "time": ["2026-09-19T00:00", "2026-09-19T01:00", "…"],
    "temperature_2m": [16.2, 15.8, "…"],
    "precipitation_probability": [0, 0, "…"],
    "wind_speed_10m": [6.1, 5.4, "…"],
    "weather_code": [0, 1, "…"],
    "is_day": [0, 0, "…"]
  }
}
```

- Les données horaires sont des **tableaux parallèles** : l'indice `i` de chaque tableau correspond à `hourly.time[i]`.
- Avec `timezone=auto`, les heures sont données à l'heure locale du lieu, **sans indication de fuseau**.
- `current.time` est l'heure courante **du lieu**. C'est elle qui permet d'écarter les heures passées, et non l'horloge du navigateur, qui peut être dans un autre fuseau.
- Une valeur peut être `null`, par exemple une probabilité de pluie non fournie.
- Vent en km/h, température en °C, probabilité en %.

### Qualité de l'air (US9)

```
GET https://air-quality-api.open-meteo.com/v1/air-quality
    ?latitude=43.61&longitude=3.88&hourly=european_aqi&timezone=auto&forecast_days=5
```

Réponse : `hourly.time` et `hourly.european_aqi`, l'indice européen, de 0 à plus de 100 ; plus il est bas, meilleur est l'air. La prévision couvre 5 jours au plus : au-delà, l'indice est inconnu.

### Codes météo (WMO)

`weather_code` suit la classification de l'Organisation météorologique mondiale. Libellés à afficher :

| Code | Signification |
|---|---|
| 0 | Ciel dégagé |
| 1, 2, 3 | Peu nuageux, partiellement nuageux, couvert |
| 45, 48 | Brouillard |
| 51, 53, 55 | Bruine |
| 56, 57 | Bruine verglaçante |
| 61, 63, 65 | Pluie faible, modérée, forte |
| 66, 67 | Pluie verglaçante |
| 71, 73, 75, 77 | Neige |
| 80, 81, 82 | Averses de pluie |
| 85, 86 | Averses de neige |
| 95 | Orage |
| 96, 99 | Orage avec grêle |

---

## 5. Exigences techniques

Chaque exigence renvoie au chapitre du cours qui la met en œuvre sur Pokedev.

### Socle

| Exigence | Chapitre |
|---|---|
| Angular 22, dernière version stable, projet créé avec la CLI ; Node 22.22.3+ ou 24.15+ | [03](03-environnement-tooling.md) |
| TypeScript strict, aucun `any` | [02](02-typescript.md) |
| Composants standalone, signaux, control flow (`@if`, `@for`…), `input()` / `output()` | [05](05-composants-signaux.md) |
| `inject()`, `@Service()` | [06](06-services-http.md) |

### Architecture

| Exigence | Chapitre |
|---|---|
| Un dossier `domain/` sans aucun import Angular : profils d'activité, score et créneaux, en fonctions pures | [09](09-etat-persistance-architecture.md) |
| Les pages récupèrent les données ; les composants d'affichage reçoivent des entrées et émettent des événements | [04](04-conception.md), [05](05-composants-signaux.md) |

### Fonctionnalités Angular

| Exigence | Chapitre |
|---|---|
| `httpResource` pour chaque appel à l'API, avec validation d'au moins une réponse (`parse` + type guard) | [06](06-services-http.md), [02](02-typescript.md) |
| Un intercepteur HTTP, par exemple : nouvelle tentative sur erreur réseau, indicateur de chargement | [06](06-services-http.md) |
| La recherche de lieu avec Signal Forms et `debounce()` ; aucune requête pour moins de 2 caractères | [10](10-formulaires.md) |
| Le routing : au moins une page de recherche, une page de prévision dont l'URL contient le lieu et l'activité, une page 404 ; les pages sont chargées à la demande | [08](08-routing.md) |
| Au moins un pipe personnalisé, par exemple le libellé d'un code météo | [07](07-directives-pipes.md) |
| Au moins une directive, ou l'utilisation de `hostDirectives` | [07](07-directives-pipes.md) |
| Pour US8 : Signal Forms avec validation croisée (température max supérieure à la température min) ; réglages persistés dans le `localStorage` et relus de manière défensive | [10](10-formulaires.md), [09](09-etat-persistance-architecture.md) |

### Qualité

| Exigence | Chapitre |
|---|---|
| Tests Vitest : 100 % de couverture sur `domain/` ; au moins un test de composant qui simule l'API avec `HttpTestingController` | [11](11-tests.md) |
| `npx ng lint` sans erreur, règles d'accessibilité comprises | [03](03-environnement-tooling.md) |
| Utilisable au clavier et sur un écran de 360 px de large | [11](11-tests.md), [12](12-securite-build-deploiement.md) |

### Sécurité et déploiement

| Exigence | Chapitre |
|---|---|
| Une Content Security Policy en production, dont le `connect-src` n'autorise que les domaines Open-Meteo utilisés | [12](12-securite-build-deploiement.md) |
| Un workflow GitHub Actions : lint, tests, build, puis déploiement sur GitHub Pages depuis `main` | [12](12-securite-build-deploiement.md) |
| Les URL profondes fonctionnent en ligne | [12](12-securite-build-deploiement.md) |

---

## 6. Jalons

Chaque jalon aboutit à un état cohérent, à committer. À la fin du jalon 6, le socle est en ligne.

1. **Le projet.** Projet créé avec la CLI, lint configuré, premier commit.
2. **Le domaine.** Modèles, profils, score et créneaux, avec leurs tests. Ni Angular ni réseau à ce stade : les exemples de la section 3 font de premiers cas de test.
3. **La prévision d'un lieu fixe.** Le service d'accès à l'API et la page de prévision pour Montpellier (43.61, 3.88), avec les 3 meilleurs créneaux (US3).
4. **La recherche et le routing.** Recherche de lieu, choix de l'activité, lien partageable (US1, US2, US5).
5. **Le détail et les états.** Détail heure par heure, pipe, directive, états de chargement, d'erreur et d'absence de créneau (US4, US6).
6. **La mise en ligne.** CSP, CI/CD, README et plan de tests.
7. **Les bonus.** User stories complémentaires, US7 à US10.

---

## 7. Ce que vous produisez

1. Un dépôt GitHub avec un historique de commits lisible : un commit par étape cohérente.
2. L'application en ligne.
3. Un `README.md` : présentation, prérequis, commandes, architecture, déploiement, licence des données. Modèle : l'exercice 1 du chapitre [12](12-securite-build-deploiement.md).
4. Un plan de tests couvrant chaque user story réalisée, avec au moins 2 tests manuels. Modèle : le chapitre [11](11-tests.md).
5. Pour vous entraîner à présenter un projet : une démonstration de 5 minutes, avec un parcours utilisateur complet puis un point technique au choix.

---

## Check-list

- [ ] Les user stories du socle, US1 à US6, fonctionnent sur un parcours complet.
- [ ] Le score et les créneaux respectent les règles métier : les tests du domaine passent, avec 100 % de couverture.
- [ ] Le chargement, l'erreur réseau et l'absence de créneau s'affichent clairement. Pour provoquer l'erreur, coupez le réseau dans les outils de développement.
- [ ] Ouvert dans un autre navigateur, le lien d'une prévision affiche la même prévision.
- [ ] `domain/` ne contient aucun import Angular.
- [ ] Chaque exigence de la section 5 se retrouve dans le code.
- [ ] Le lint et les tests passent : la CI est verte.
- [ ] L'application est en ligne, avec la CSP, sans erreur dans la console.
- [ ] L'application se parcourt au clavier ; le lint d'accessibilité et Lighthouse ne relèvent aucun problème.
- [ ] La mention de la licence Open-Meteo est visible.
- [ ] Le README et le plan de tests sont complets.

Une fois le projet terminé, comparez vos choix avec la réalisation fournie dans `time-to-corrige.zip`.

---

[← Chapitre précédent](12-securite-build-deploiement.md) · [Sommaire](README.md)
