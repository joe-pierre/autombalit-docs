# CONVENTIONS.md — Règles de codage

## Langue

- **Domaine métier (modèles Django, champs, endpoints, messages MQTT)** : en français, cohérent avec la conception déjà validée (`Camion`, `Chauffeur`, `Tournee`, `PositionCamion`...). Ne pas renommer en anglais sans décision explicite dans `DECISIONS.md`.
- **Code générique (fonctions utilitaires, noms de variables techniques, commentaires de code)** : anglais standard, sauf ambiguïté où le français est plus clair pour un terme métier.
- **Documentation projet** (`SPEC.md`, `DECISIONS.md`, `TODO.md`, commit messages) : français.
- **Dart/Flutter** : conventions Dart standard (anglais pour les noms techniques), textes UI en français (le public cible est sénégalais francophone).

## Nommage

- **Python/Django** : `snake_case` pour variables/fonctions/fichiers, `PascalCase` pour les classes (modèles, serializers, views).
- **Champs de modèle** : toujours en français, cohérents avec le schéma déjà défini (ex. `horodatage`, pas `timestamp`).
- **Endpoints API** : `kebab-case`, pluriel, préfixés `/api/` (ex. `/api/points-enregistres/`).
- **Topics MQTT** : `camions/{camion_id}/position` — namespace stable, ne pas modifier sans mettre à jour l'ACL Mosquitto en conséquence.
- **Groupes WebSocket (Channels)** : `camion_<uuid>` ou `zone_<uuid>`.
- **Dart/Flutter** : `camelCase` pour variables/fonctions, `PascalCase` pour classes/widgets, fichiers en `snake_case.dart`.
- **Branches Git** : `<type>/<sujet-court>` (ex. `feat/positions-mqtt`, `fix/eta-seuils`).

## Réponses API

Format JSON standard pour toutes les réponses DRF :

```json
{
  "success": true,
  "data": { ... },
  "error": null
}
```

En cas d'erreur :

```json
{
  "success": false,
  "data": null,
  "error": { "code": "VALIDATION_ERROR", "message": "..." }
}
```

- Codes HTTP cohérents avec la sémantique REST (201 création, 204 suppression, 400 validation, 401/403 auth, 404 introuvable, 409 conflit).
- Toujours paginer les listes potentiellement longues (`PositionCamion`, `Signalement`) avec la pagination DRF standard.

## Validation

- Toute donnée entrante géospatiale doit être validée en `EPSG:4326` avant stockage (rejet explicite sinon, pas de reprojection silencieuse).
- Les serializers DRF sont responsables de la validation métier (ex. un `PointEnregistre` doit tomber dans une `Zone` existante pour être exploitable — sinon `zone` reste `null` et l'utilisateur est informé).
- Validation des GeoJSON à l'upload : type de géométrie attendu selon le contexte (`LineString`/`MultiLineString` pour une tournée, `Polygon` pour une zone), présence des `properties` requises (nom, zone, jours de collecte).
- Aucune validation de logique métier critique côté client uniquement (Flutter) — toujours revalidée côté backend.

## Broadcasting

- Un seul canal de diffusion temps réel vers le citoyen : Django Channels / WebSocket, à la demande (écran carte live ouvert uniquement).
- Le backend est l'unique point d'entrée MQTT → traitement → diffusion. Aucun client (citoyen) ne se connecte directement au broker MQTT.
- Diffusion par groupe (`zone_<id>` ou `camion_<id>`), jamais de broadcast global à tous les clients connectés.
- Une position ne déclenche une diffusion WebSocket que si elle est associée à une assignation `en_cours`.

## Tests

- `pytest-django` comme framework de test backend.
- `factory_boy` pour les fixtures de modèles (Camion, Chauffeur, Tournee, etc.).
- Couverture minimale attendue sur la logique métier critique : calcul ETA, logique de seuils/notifications, validation GeoJSON, ACL MQTT. Les vues CRUD simples peuvent avoir une couverture plus légère.
- Tests d'intégration pour le flux complet position MQTT → ETA → notification, avec mocks sur OSRM et FCM (ne jamais appeler les services externes réels en test).
- Flutter : tests unitaires sur la logique de queue hors-ligne (chauffeur) et le parsing des notifications (citoyen) au minimum.

## Architecture et paradigmes

Objectif : un code lisible et maintenable dans la durée, pas de fichier fourre-tout, une logique métier qui ne dépend pas d'un endroit unique pour être comprise.

- **Service layer obligatoire pour toute logique métier non triviale.** Les vues/serializers DRF ne contiennent pas de logique métier — ils valident l'entrée, appellent un service, formatent la sortie. Toute règle métier décrite dans `SPEC.md` §4 (calcul ETA, seuils de notification, validation GeoJSON, gestion des positions en rafale, ACL MQTT) vit dans une fonction/classe de service dédiée, testable indépendamment du framework web.
- **Structure par app** : chaque app Django (`core`, `tracking`, `citizens`, `notifications`, `realtime`, `geo_import`) contient au besoin `services.py` (ou un sous-module `services/` si le fichier grossit), `selectors.py` (lectures/requêtes complexes), `models.py` (structure des données uniquement, pas de logique métier lourde dans les méthodes de modèle au-delà de propriétés simples), `serializers.py`, `views.py`, `tests/`.
- **Pas de "god file"** : dès qu'un fichier (`views.py`, `services.py`, `models.py`) dépasse ~300-400 lignes ou mélange plusieurs responsabilités non liées, le découper par sous-domaine (ex. `tracking/services/eta.py`, `tracking/services/mqtt_handler.py`, `tracking/services/thresholds.py` plutôt qu'un unique `tracking/services.py` monolithique).
- **Pas de "god model"** : un modèle ne doit pas accumuler des méthodes couvrant plusieurs responsabilités (ex. `PositionCamion` ne doit pas contenir la logique de calcul d'ETA ni la logique de déclenchement de notification — ces responsabilités vivent dans des services dédiés qui consomment le modèle).
- **Séparation lecture/écriture** : les requêtes de lecture complexes (agrégats, filtres géospatiaux) passent par des `selectors`/`queries` dédiés plutôt que d'être dupliquées dans plusieurs vues.
- **Injection de dépendances légère** : les intégrations externes (OSRM, FCM, MQTT, Firebase Auth) sont chacune encapsulées dans un client dédié (`tracking/clients/osrm_client.py`, `notifications/clients/fcm_client.py`, etc.), jamais appelées directement depuis une vue — ça permet de les mocker facilement en test (voir §Tests) et de les remplacer sans toucher à la logique métier.
- **Un fichier = une responsabilité** : éviter qu'un même fichier mélange, par exemple, la gestion des seuils de notification et l'appel FCM — deux services distincts, composés l'un par l'autre.
- **Flutter** : même logique — pas de logique métier dans les widgets. Séparer `services/` (appels API, MQTT, géolocalisation), `repositories/` (accès aux données locales/cache hors-ligne), `providers`/state management, et `widgets/` (affichage pur). Un widget ne fait pas d'appel réseau directement.
- **Revue systématique** : à chaque tâche livrée par Claude Code, vérifier qu'aucun fichier n'est devenu un god file avant de committer — c'est un critère de relecture, pas seulement une préférence de style.

## Sécurité

- Aucun secret (clés Firebase, identifiants MQTT, `SECRET_KEY` Django) en dur dans le code — variables d'environnement uniquement, `.env` non commité.
- JWT à durée de vie courte, refresh token géré côté Flutter via stockage sécurisé (`flutter_secure_storage`), jamais en `localStorage`/stockage non chiffré.
- ACL Mosquitto stricte : un chauffeur ne peut publier que sur son propre topic.
- Toute route d'administration (`/api/admin/...`) protégée par permission de rôle (`admin société` ou `super admin`), jamais accessible à un compte citoyen/chauffeur simple.
- Rate limiting sur les endpoints publics sensibles (signalements, création de compte) pour limiter les abus.

## Partials / Frontend (Web Admin)

- Choix du framework frontend du web admin non tranché à ce stade (Django templates classiques vs templates + htmx pour l'interactivité de la carte). À décider en Phase 4 et consigné dans `DECISIONS.md` avant implémentation.
- Si templates Django : un template de base + partials par section (upload GeoJSON, gestion tournées, validation chauffeurs, dashboard signalements), pas de logique métier dans les templates.
- Carte interactive côté web : Leaflet (cohérent avec l'approche OSM gratuite déjà retenue côté mobile), pas de Mapbox/Google Maps payant.
