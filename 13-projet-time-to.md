[← 12 · Sécurité, build et déploiement](12-securite-build-deploiement.md) · [Sommaire](README.md)

---

# Phase 12 — Projet Time To

Réalisez **seul**, de la création du projet au déploiement, l'application Time To. Ce document est le cahier des charges.

---

## 1. Le besoin

Time To répond à une question : **« Quand puis-je faire mon activité en extérieur cette semaine ? »**

L'utilisateur choisit un lieu et une activité (vélo, randonnée, course à pied, étendre le linge). L'application récupère les prévisions météo heure par heure sur 7 jours, attribue à chaque heure un score de 0 à 100 selon l'activité, puis affiche les meilleurs créneaux.

L'application n'a pas de backend : elle interroge directement l'API publique Open-Meteo depuis le navigateur.

---

## 2. User stories

### Obligatoires

| Id | User story |
|---|---|
| US1 | En tant qu'utilisateur, je veux chercher un lieu par son nom et choisir parmi les résultats, afin de consulter ses prévisions. |
| US2 | En tant qu'utilisateur, je veux choisir une activité, afin que le score corresponde à ce que je veux faire. |
| US3 | En tant qu'utilisateur, je veux voir les 3 meilleurs créneaux de la semaine (début, fin, score moyen, températures), afin de planifier ma sortie. |
| US4 | En tant qu'utilisateur, je veux voir le détail heure par heure d'une journée (heure, conditions, température, vent, pluie, score), afin de comprendre le score. |
| US5 | En tant qu'utilisateur, je veux partager un lien vers une prévision (lieu et activité), afin qu'une autre personne voie la même chose. |
| US6 | En tant qu'utilisateur, je veux être informé clairement pendant le chargement, en cas d'erreur réseau ou si aucun créneau n'est trouvé. |

### Complémentaires

| Id | User story |
|---|---|
| US7 | En tant qu'utilisateur, je veux utiliser ma position actuelle, afin de ne rien saisir. |
| US8 | En tant qu'utilisateur, je veux ajuster les seuils d'une activité (températures, vent, pluie), afin que le score corresponde à mes préférences. Les réglages sont conservés après rechargement. |
| US9 | En tant qu'utilisateur sensible à la pollution, je veux que la qualité de l'air soit prise en compte pour la randonnée et la course. |
| US10 | En tant qu'utilisateur, je veux retrouver le dernier lieu consulté en ouvrant l'application. |

---

## 3. Règles métier

### Profils d'activité par défaut

| Activité | Température idéale | Tolérance | Vent max | Pluie max | Journée seulement | Qualité de l'air max |
|---|---|---|---|---|---|---|
| Vélo | 15 à 25 °C | 10 °C | 30 km/h | 30 % | oui | — |
| Randonnée | 12 à 24 °C | 10 °C | 40 km/h | 30 % | oui | 60 |
| Course à pied | 8 à 20 °C | 10 °C | 35 km/h | 40 % | non | 40 |
| Étendre le linge | 18 à 35 °C | 12 °C | 45 km/h | 10 % | oui | — |

### Score d'une heure

Si l'activité se pratique en journée seulement et qu'il fait nuit, le score vaut 0. Sinon :

```
score = arrondi(100 × facteurTempérature × facteurVent × facteurPluie × facteurAir)
```

| Facteur | Règle |
|---|---|
| Température | 1 dans la plage idéale ; hors de la plage, diminue linéairement jusqu'à 0 à un écart égal à la tolérance |
| Vent | 1 jusqu'à la moitié du vent max ; diminue ensuite linéairement jusqu'à 0 au vent max |
| Pluie | 0 au-delà de la pluie max ; sinon `1 − probabilité / 200` ; 0,8 si la probabilité est inconnue |
| Air | 1 si l'indice est inconnu ou si l'activité n'a pas de seuil ; 1 sous le seuil ; 0,3 au-dessus |

Exemples pour le vélo : 20 °C → facteur 1 ; 10 °C → 0,5 ; 5 °C → 0. Vent de 15 km/h → 1 ; 22,5 km/h → 0,5.

### Créneaux

Un créneau est une suite d'**heures consécutives** dont le score est **au moins 60**, d'une durée d'**au moins 2 heures**. Les créneaux sont triés par score moyen décroissant, puis par durée décroissante. On affiche les 3 premiers. Les heures déjà passées sont ignorées.

---

## 4. L'API Open-Meteo

Gratuite pour un usage non commercial, sans clé ni compte, appelable depuis un navigateur (CORS autorisé). Limite : 10 000 appels par jour. Licence des données : **CC BY 4.0** ; la mention « Données météo : Open-Meteo (CC BY 4.0) » avec un lien vers https://open-meteo.com est **obligatoire** dans l'application.

Documentation : https://open-meteo.com/en/docs

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
- Une valeur peut être `null` (par exemple une probabilité de pluie non fournie).
- Vent en km/h, température en °C, probabilité en %.

### Qualité de l'air

```
GET https://air-quality-api.open-meteo.com/v1/air-quality
    ?latitude=43.61&longitude=3.88&hourly=european_aqi&timezone=auto&forecast_days=5
```

Réponse : `hourly.time` et `hourly.european_aqi` (indice européen, de 0 à plus de 100 ; plus il est bas, meilleur est l'air). La prévision couvre 5 jours au plus : au-delà, l'indice est inconnu.

### Codes météo (WMO)

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

## 5. Contraintes techniques

**Socle**
- Angular 22 (dernière version stable), projet créé avec la CLI, Node 22.22.3+ ou 24.15+.
- Composants standalone, signaux, control flow (`@if`, `@for`…), `inject()`, `input()` / `output()`, `@Service()`.
- TypeScript strict, aucun `any`.

**Architecture**
- Un dossier `domain/` sans aucun import Angular : profils d'activité, score, créneaux, en fonctions pures.
- Les pages récupèrent les données ; les composants d'affichage reçoivent des entrées et émettent des événements.

**Fonctionnalités Angular à mettre en œuvre**
- `httpResource` pour les trois appels, avec validation d'au moins une réponse (`parse` + type guard).
- Un intercepteur HTTP (par exemple : nouvelle tentative sur erreur réseau, indicateur de chargement).
- La recherche de lieu avec Signal Forms et `debounce()` ; aucune requête pour moins de 2 caractères.
- Le routing : au moins une page de recherche, une page de prévision dont l'URL contient le lieu et l'activité, une page 404 ; chargement différé des pages.
- Au moins un pipe personnalisé (par exemple : libellé d'un code météo).
- Au moins une directive ou l'utilisation de `hostDirectives`.
- Si US8 est réalisée : Signal Forms avec validation croisée (température max supérieure à la température min) et persistance dans le `localStorage`, lue de manière défensive.

**Qualité**
- Tests Vitest : 100 % de couverture sur `domain/` ; au moins un test de composant qui simule l'API avec `HttpTestingController`.
- `npx ng lint` sans erreur, règles d'accessibilité comprises.
- Utilisable au clavier et sur un écran de 360 px de large.

**Sécurité et déploiement**
- Une Content Security Policy en production, qui n'autorise que les trois domaines Open-Meteo en `connect-src`.
- Un workflow GitHub Actions : lint, tests, build, puis déploiement sur GitHub Pages depuis `main`.
- Les URL profondes fonctionnent en ligne.

---

## 6. Livrables

1. L'URL du dépôt GitHub, avec un historique de commits lisible (un commit par étape cohérente).
2. L'URL de l'application en ligne.
3. Un `README.md` : présentation, prérequis, commandes, architecture, déploiement, licence des données.
4. Un plan de tests couvrant chaque user story réalisée, avec au moins 2 tests manuels.
5. Une démonstration de 5 minutes : un parcours utilisateur complet, puis un point technique au choix.

---

## 7. Jalons proposés

1. Projet créé, lint configuré, premier commit.
2. Domaine : modèles, profils, score, créneaux, et leurs tests.
3. Service API et page de prévision pour un lieu fixe (Montpellier : 43.61, 3.88).
4. Recherche de lieu et routing.
5. Détail heure par heure, pipe, directive, états de chargement et d'erreur.
6. User stories complémentaires.
7. CSP, CI/CD, README, plan de tests.

---

## 8. Critères de validation

| Critère | Vérification |
|---|---|
| Les user stories obligatoires fonctionnent | Démonstration |
| Le score et les créneaux respectent les règles métier | Tests du domaine, couverture 100 % |
| Les états de chargement, d'erreur et d'absence de résultat sont gérés | Démonstration (réseau coupé dans les outils de développement) |
| Le lien d'une prévision est partageable | Ouverture de l'URL dans un autre navigateur |
| Le code respecte l'architecture demandée | Lecture du code : `domain/` sans import Angular |
| Les éléments Angular demandés sont présents | Lecture du code |
| Le lint et les tests passent | CI verte |
| L'application est en ligne avec la CSP, sans erreur de console | Visite de l'URL publique |
| L'application est accessible | Navigation au clavier, lint d'accessibilité, Lighthouse |
| La mention de la licence Open-Meteo est visible | Visite |
| La documentation et le plan de tests sont complets | Lecture |

Une réalisation possible est fournie dans `time-to-corrige.zip`.

---

[← 12 · Sécurité, build et déploiement](12-securite-build-deploiement.md) · [Retour au sommaire](README.md)
