# Décisions techniques et bugs résolus

## [CHOIX] Domicile enregistré plutôt que tracking citoyen en continu

**Contexte :** besoin de notifier le citoyen selon l'ETA du camion sans connaître sa position en temps réel.
**Alternatives :** tracking GPS continu du citoyen vs enregistrement d'un point fixe (domicile).
**Décision :** le citoyen enregistre un ou plusieurs points fixes (domicile) ; seule la position du camion est transmise en continu.
**Leçon :** meilleur pour la batterie, la vie privée et la complexité côté client.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tracé de tournée = référence indicative, pas un chemin strict

**Contexte :** le trajet réel du camion varie selon le chauffeur/les circonstances.
**Décision :** le GeoJSON de la tournée sert à rattacher zones et calendrier, pas de map-matching strict pour l'ETA. L'ETA repose sur OSRM (distance routière réelle) + facteur de correction historique par zone.
**Statut :** 🔵 Choix assumé

## [CHOIX] Backend : Django + GeoDjango plutôt que Symfony/Laravel

**Contexte :** besoin d'un backend avec support géospatial fort (PostGIS), API REST, et proximité avec un futur volet data/statistiques.
**Alternatives :** Symfony (API Platform), Laravel (packages spatiaux tiers moins matures).
**Décision :** Django + GeoDjango + Django REST Framework, pour l'intégration ORM native des requêtes spatiales et la maturité de l'écosystème SIG en Python.
**Statut :** 🔵 Choix assumé

## [CHOIX] Import géographique : GeoJSON pré-converti par l'équipe SIG

**Contexte :** éviter de maintenir un pipeline de conversion Shapefile/KMZ → GeoJSON côté backend.
**Décision :** l'équipe SIG convertit en amont (QGIS/ArcGIS exportent nativement en GeoJSON) et upload directement le GeoJSON via le web admin. Le backend valide et stocke, il ne transforme pas.
**Contrainte imposée à l'équipe SIG :** SRID EPSG:4326 obligatoire, convention de `properties` définie (nom, zone, jours de collecte, id camion assigné).
**Statut :** 🔵 Choix assumé

## [CHOIX] Routing : OSRM self-hosted

**Contexte :** besoin d'un moteur de calcul d'itinéraire gratuit et sans quota, vu le volume de requêtes attendu (recalcul à chaque position proche d'une zone).
**Alternatives évaluées :** GraphHopper (cloud limité), Valhalla, OpenRouteService (quota cloud).
**Décision :** OSRM self-hosted avec données OSM Sénégal via Geofabrik. Gratuit, illimité, pas de dépendance tierce.
**Point de vigilance :** qualité variable des données OSM selon les quartiers — à vérifier avant de dépendre uniquement du routing sur les zones cibles.
**Statut :** 🔵 Choix assumé

## [CHOIX] Communication chauffeur → backend : MQTT plutôt que HTTP polling ou WebSocket

**Contexte :** connectivité mobile instable au Sénégal, besoin d'économiser batterie/data, besoin de résister aux coupures réseau.
**Alternatives :** HTTP polling (overhead de connexion répété), WebSocket (gestion de reconnexion plus lourde à maintenir côté mobile).
**Décision :** MQTT (broker Mosquitto self-hosted), QoS 1/2 pour garantir la livraison malgré les coupures, mode publish/subscribe adapté au cas d'usage.
**Statut :** 🔵 Choix assumé

## [CHOIX] Carte live citoyen : WebSocket à la demande (Django Channels + Redis)

**Contexte :** besoin d'un canal temps réel uniquement quand l'écran carte est ouvert côté citoyen, sans connexion permanente en arrière-plan.
**Décision :** WebSocket via Django Channels, activé à l'ouverture de l'écran carte, fermé à la sortie. Le citoyen ne se connecte jamais directement au broker MQTT — seul le backend relaie.
**Statut :** 🔵 Choix assumé

## [CHOIX] Notifications : Firebase Cloud Messaging

**Contexte :** besoin d'un service de notifications push gratuit sans limite de volume.
**Décision :** FCM, gratuit même à fort volume, cohérent avec l'usage déjà prévu de Firebase Auth.
**Statut :** 🔵 Choix assumé

## [CHOIX] Authentification citoyen et chauffeur : Firebase Auth (OTP téléphone)

**Contexte :** besoin d'une authentification simple sans mot de passe, cohérente avec le reste de la stack Firebase.
**Décision :** Firebase Auth par OTP SMS pour les deux profils. Côté chauffeur, ajout d'un statut de validation manuelle (`en_attente/valide/rejete`) par un admin société avant toute publication de position possible.
**Statut :** 🔵 Choix assumé

## [CHOIX] Mobile : Flutter plutôt que React Native ou natif

**Contexte :** besoin d'une codebase unique pour deux apps (chauffeur/citoyen) et deux plateformes, avec un tracking GPS en arrière-plan fiable.
**Décision :** Flutter — plugins matures pour le tracking GPS en tâche de fond, performance native compilée, `flutter_map` gratuit sans clé API. Android priorisé pour le MVP (marché sénégalais majoritairement Android), iOS repoussé en V2.
**Statut :** 🔵 Choix assumé

## [CHOIX] Modèle camion ↔ tournée ↔ chauffeur : assignation journalière plutôt que relation fixe

**Contexte :** un camion peut changer de tournée d'un jour à l'autre (remplacement, réorganisation).
**Décision :** table `AssignationJournaliere (camion, chauffeur, tournee, date)` plutôt qu'un champ `camion_id` fixe sur `Tournee`. Contrainte unique `(camion, date)`.
**Statut :** 🔵 Choix assumé

## [CHOIX] Relation Zone ↔ Tournée : many-to-many avec ordre de passage

**Contexte :** une tournée traverse souvent plusieurs quartiers dans un ordre donné.
**Décision :** table de liaison `TourneeZone (tournee, zone, ordre_passage)` plutôt qu'une relation simple un-à-plusieurs.
**Statut :** 🔵 Choix assumé

## [CHOIX] Calcul ETA : OSRM + facteur de correction historique par zone

**Contexte :** le temps de trajet théorique (routing pur) sous-estime systématiquement l'ETA réel à cause des arrêts fréquents de collecte.
**Décision :** combiner distance/temps OSRM avec un facteur de correction empirique par zone, calculé à partir de l'historique `PositionCamion` (temps observé vs temps théorique), affiné automatiquement avec l'accumulation de données. Valeur par défaut tant que l'historique est insuffisant.
**Statut :** 🔵 Choix assumé

## [CHOIX] Anti-spam des notifications par seuil

**Contexte :** un camion à l'arrêt prolongé peut faire osciller l'ETA autour d'un seuil et déclencher des notifications répétées.
**Décision :** une notification par seuil (30/20/10/5 min) par `(utilisateur, camion_du_jour)` par jour, reset quotidien. Pas de re-notification si l'ETA remonte puis redescend sous un seuil déjà notifié.
**Statut :** 🔵 Choix assumé

## [CHOIX] Rétention des positions : historique limité + agrégats

**Contexte :** éviter une croissance illimitée des données de position brutes, réduire le risque en cas de fuite de données.
**Décision :** historique brut `PositionCamion` conservé 24-48h, puis uniquement des agrégats statistiques anonymisés (heure moyenne de passage par zone/jour) conservés à long terme.
**Statut :** 🔵 Choix assumé

## [CHOIX] Positions en rafale après coupure réseau : ne traiter que la plus récente

**Contexte :** après une coupure, le chauffeur envoie un lot de positions accumulées localement ; les traiter toutes comme du temps réel déclencherait des notifications incohérentes.
**Décision :** seule la position au timestamp le plus récent du lot déclenche le recalcul ETA/notification ; les autres sont stockées pour historique/statistiques uniquement.
**Statut :** 🔵 Choix assumé

## [CHOIX] Environnement de développement local : Docker Compose

**Contexte :** besoin d'un environnement local reproductible pour PostGIS, Redis, Mosquitto et OSRM sans installation manuelle de chaque service sur la machine de dev.
**Décision :** un unique `docker-compose.yml` orchestrant `db` (PostGIS), `redis`, `mosquitto`, `osrm`, `backend` (Django) et `mqtt_listener`. `docker compose up` suffit à démarrer toute la stack locale. Application web et base de données doivent être pleinement exploitables en local via cette stack avant toute mise en production.
**Statut :** 🔵 Choix assumé

## [CHOIX] Test mobile local : appareil Android physique (Samsung) via USB plutôt qu'émulateur

**Contexte :** besoin de tester les apps Flutter (chauffeur/citoyen) dans des conditions proches du réel, notamment GPS et notifications FCM.
**Décision :** développement et tests mobiles effectués sur un téléphone Samsung physique connecté en USB (débogage USB activé), plutôt que sur un émulateur Android. Connexion à l'API locale via IP locale du même réseau Wi-Fi, ou via `adb reverse tcp:8000 tcp:8000` si le partage réseau n'est pas disponible.
**Statut :** 🔵 Choix assumé

## [CHOIX] Dépôt de documentation séparé (`autombalit-docs`)

**Contexte :** deux dépôts de code distincts (`autombalit-backend`, `autombalit-mobile`) ; les fichiers de suivi (`SPEC.md`, `TODO.md`, `DECISIONS.md`, `TASK_PROMPTS.md`...) couvrent les deux à la fois.
**Alternatives évaluées :** dupliquer les fichiers dans chaque dépôt vs les centraliser dans le dépôt backend vs un dépôt dédié.
**Décision :** dépôt Git séparé `autombalit-docs`, au même niveau que `autombalit-backend`/`autombalit-mobile` dans un dossier parent commun. Évite la duplication et la divergence entre deux copies. Les sessions Claude Code se lancent depuis le dossier parent pour avoir les trois dépôts visibles simultanément.
**Statut :** 🔵 Choix assumé

## [CHOIX] Nom du projet : AutoMbalit (anciennement Geopoubelle)

**Contexte :** "Geopoubelle" était un nom de travail pour la conception, pas destiné à être le nom public de l'app. Recherche d'un nom plus original, avec une couleur locale/wolof.
**Alternatives évaluées :** pistes wolof pur (Waxtu, Fanal), jeux de mots FR/wolof (Bipoubelle, Kaay Poubelle, Mbalit Waxtu, Kaay Mbalit), noms neutres internationaux (Arrivo, Passago, Tourné), pistes centrées sur le métier plutôt que l'objet (Borom Mbalit, Tuurkat — termes wolof attestés pour "éboueur").
**Décision :** **AutoMbalit** — "mbalit" confirmé comme signifiant "poubelle/ordures" en wolof (sources : glossaire genre et assainissement au Sénégal, sophielehire.com). Combinaison retenue par choix personnel plutôt que sur un terme 100% attesté tel quel — l'association "Auto" + "Mbalit" est une construction, pas une expression figée du wolof.
**Renommage effectué :** tous les fichiers `.md` du projet (dépôts, packages, identifiants techniques : `autombalit-backend`, `autombalit-mobile`, `autombalit_backend`, `com.autombalit`, bases `autombalit_dev`/`autombalit_prod`, etc.).
**Point de vigilance :** une recherche a fait remonter une occurrence du terme dans un texte de rap sénégalais avec une connotation dépréciative — signalé et discuté avec l'utilisateur, qui confirme que le nom sonne bien à l'oreille. Validation linguistique considérée comme faite sur cette base ; reste à vérifier la disponibilité du nom de domaine, des comptes réseaux sociaux, des stores et d'une éventuelle marque OAPI avant lancement public (voir `TODO.md` Phase 0).
**Statut :** 🔵 Choix assumé — disponibilité domaine/réseaux sociaux/stores/marque en attente de vérification

## [CHOIX] Direction visuelle : coloré et accessible

**Contexte :** identité visuelle non définie initialement (`SPEC.md` §10 marqué "à remplir"), besoin de trancher avant la Phase 3 (apps mobiles).
**Alternatives évaluées :** sobre/institutionnel (mairie, service public) vs coloré/accessible (grand public, tous âges).
**Décision :** direction coloré/accessible — vert dominant pour l'app citoyen, ambre/orange pour l'app chauffeur, rouge réservé aux alertes/ETA imminent. Typographie généreuse, icônes pleines toujours doublées de texte, zones tactiles larges, contraste WCAG AA minimum. Détail complet dans `SPEC.md` §10.
**Statut :** 🔵 Choix assumé

## [CHOIX] Glassmorphism en usage mesuré, jamais sur les éléments critiques

**Contexte :** envie d'ajouter un effet glassmorphism à la direction visuelle coloré/accessible déjà retenue, en tension avec la contrainte de contraste WCAG AA (public cible incluant personnes âgées/faible littératie numérique).
**Décision :** glassmorphism autorisé uniquement sur des éléments secondaires (panneau flottant carte live, barres de navigation), jamais sur les éléments porteurs d'information critique (ETA, calendrier) ou d'action (boutons). Détail des paramètres dans `SPEC.md` §10.
**Statut :** 🔵 Choix assumé

## [CHOIX] Version Django/Python (Tâche 1) : Django 6.1.1 / Python 3.13 plutôt que Django 5.x / Python 3.11+

**Contexte :** `SPEC.md` §2 et `TASK_PROMPTS.md` (Tâche 1, Étape 0) recommandaient Django 5.x / Python 3.11+. Le scaffolding manuel réalisé en amont (`GUIDE_DU_DEVELOPPEUR.md` Phase 1.3, avant la Tâche 1) avait installé sans version figée les dernières versions disponibles au moment de l'exécution : Django 6.1.1 et Python 3.13.0 (venv local `autombalit-backend/venv`).
**Alternatives :** revenir sur Django 5.x LTS / Python 3.11+ pour coller à `SPEC.md` (implique de recréer le venv et de regénérer `requirements.txt`) vs conserver les versions déjà scaffoldées et commitées.
**Décision :** conserver Django 6.1.1 / Python 3.13 — confirmé explicitement par l'utilisateur avant de coder (Tâche 1, Étape 0, `AskUserQuestion`). Aucune régression identifiée : `python manage.py check` et `migrate` passent sans erreur sur PostGIS avec cette version.
**Leçon :** à l'avenir, figer les versions Django/Python dès le premier `pip install` (ex. `pip install "django>=5,<6"`) plutôt que d'attraper la dernière version disponible, pour éviter l'écart avec `SPEC.md` constaté ici.
**Statut :** 🔵 Choix assumé — `SPEC.md` §2 reste à corriger dans une prochaine passe documentaire pour refléter Django 6.1.1 / Python 3.13 au lieu de "Django 5.x".

## [CHOIX] Modèles Tâche 2 : valeurs de `AssignationJournaliere.statut` et `CalendrierCollecte.jour_semaine`

**Contexte :** `SPEC.md` §3 définit les champs `statut` (`AssignationJournaliere`) et `jour_semaine` (`CalendrierCollecte`) sans préciser leurs valeurs possibles ; ce niveau de détail n'était pas non plus explicite dans l'historique de conception disponible dans ce dépôt.
**Décision :**
- `AssignationJournaliere.statut` : `planifiee` / `en_cours` / `terminee` — cohérent avec l'événement `tournee.status` décrit dans `SPEC.md` §6 (`en_cours`, `terminee`), avec ajout d'un état initial `planifiee` avant démarrage de la tournée.
- `CalendrierCollecte.jour_semaine` : entier 0 (lundi) à 6 (dimanche) avec choices Django explicites, plutôt qu'un `CharField` libre.
- Contrainte additionnelle non explicitement demandée mais ajoutée pour l'intégrité des données : `unique_together` sur `TourneeZone (tournee, zone)` (une zone ne peut apparaître qu'une fois par tournée) et sur `CalendrierCollecte (tournee, jour_semaine)` (un seul horaire par jour pour une tournée donnée).
**Statut :** 🔵 Choix assumé — à réviser si l'historique de conception original prévoyait des valeurs différentes.

## [RÉSOLU] Tâche 3 (Flutter) : versions Gradle/AGP/Kotlin incompatibles avec le Flutter SDK installé

**Contexte :** lors de la vérification de la Tâche 3 (flavors `citoyen`/`chauffeur`), `flutter build apk` échouait sur le scaffolding initial (`GUIDE_DU_DEVELOPPEUR.md` §2.1), avant même toute logique de flavor.
**Symptôme / Problème :** échecs en cascade : Gradle 8.12 < minimum 8.14 requis par le plugin Gradle de Flutter 3.47.3, puis AGP 8.7.3 < minimum 8.11.1, puis Kotlin 2.1.0 < minimum 2.2.20.
**Cause :** le scaffolding avait été généré par `flutter create` sans versions figées ; les versions par défaut du template au moment du scaffolding sont devenues incompatibles avec le SDK Flutter effectivement installé sur la machine de dev (Flutter 3.47.3), non lié aux flavors ajoutés dans cette tâche (confirmé via `git log` sur `android/gradle/wrapper/gradle-wrapper.properties`, seul commit = scaffolding initial).
**Fix :** relevé Gradle → 8.14, AGP → 8.11.1, Kotlin (`org.jetbrains.kotlin.android`) → 2.2.20 dans `android/settings.gradle.kts` et `android/gradle/wrapper/gradle-wrapper.properties`. `android/gradle.properties` a reçu deux flags ajoutés automatiquement par l'outil de migration Flutter (`android.builtInKotlin=false`, `android.newDsl=false`) pour rester compatible avec la structure `build.gradle.kts` existante sans migration complète vers AGP 9 (DSL différent, hors scope de cette tâche).
**Vérification :** `flutter build apk --flavor citoyen` et `--flavor chauffeur` réussissent ; les deux APK installées et lancées sur le Samsung physique via `adb`, chacune affichant son écran placeholder distinct (`App Citoyen` / `App Chauffeur`) avec un `applicationId` distinct confirmé dans le manifest fusionné (`com.autombalit.citoyen` / `com.autombalit.chauffeur`).
**Point de vigilance :** des warnings (non bloquants) subsistent invitant à migrer vers AGP 9.0.1+/Gradle 9.1+/Kotlin 2.3.20+ « bientôt » — reporté volontairement car AGP 9 impose une nouvelle DSL Gradle qui casserait la config actuelle ; à traiter dans une tâche dédiée plutôt qu'en aparté d'une tâche de flavors.
**Leçon :** comme pour Django/Python (voir plus bas), figer les versions d'outillage dès le scaffolding initial (`flutter create`) éviterait ce type de dérive silencieuse découverte seulement au moment de builder pour de vrai.
**Statut :** ✅ Résolu

## [CHOIX] Tâche 4 : serializers/viewsets centralisés dans l'app `api`

**Contexte :** `Zone` et `Tournee` vivent dans l'app `core`, mais `SPEC.md` §5 décrit l'app `api` comme portant « serializers DRF, routers, permissions par rôle ». Deux options possibles : centraliser dans `api`, ou répartir serializers/vues dans l'app métier propriétaire du modèle.
**Décision :** confirmé par l'utilisateur — tout le code DRF (serializers, viewsets, routers) reste dans l'app `api`, qui importe les modèles de `core`. Les apps métier (`core`, `tracking`, etc.) restent limitées aux modèles/services/selectors/clients. Permission par défaut : `IsAuthenticated` uniquement, pas de granularité par rôle à ce stade (auth Firebase/rôles prévue Tâche 12+).
**Statut :** 🔵 Choix assumé

## [CHOIX] Format de réponse JSON standard implémenté au niveau du renderer/exception handler

**Contexte :** `CONVENTIONS.md` §Réponses API impose `{"success", "data", "error"}` pour toutes les réponses DRF, mais ce mécanisme n'existait pas encore — la Tâche 4 est la première à exposer de vrais endpoints DRF.
**Décision :** implémenté globalement plutôt que par vue, via `api/renderers.py` (`StandardJSONRenderer`, enveloppe toute réponse non-`None`) et `api/exceptions.py` (`standard_exception_handler`, normalise les exceptions DRF en `{"code", "message"}`), branchés dans `REST_FRAMEWORK.DEFAULT_RENDERER_CLASSES`/`EXCEPTION_HANDLER` (`settings.py`). S'applique donc automatiquement à tous les futurs endpoints, y compris `/api/token/` (simplejwt) sans code spécifique à écrire pour chacun.
**Point de vigilance :** `response.data` (utilisé dans les tests DRF) reste la donnée *avant* enveloppe — seul `response.content`/`response.json()` reflète le format final `{"success","data","error"}`. Les tests de la Tâche 4 utilisent `response.json()` pour cette raison.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 4 : `TourneeSerializer.zones` en lecture seule, pas de CRUD `TourneeZone` dans cette tâche

**Contexte :** `Tournee.zones` est un many-to-many via `TourneeZone` qui porte un champ obligatoire (`ordre_passage`), non gérable par un simple `PrimaryKeyRelatedField` en écriture sans logique métier additionnelle.
**Décision :** `zones` exposé en lecture seule (liste des ids) sur `TourneeSerializer` ; la création/modification des associations `TourneeZone` (avec ordre de passage) n'est pas couverte par cette tâche — hors périmètre annoncé (« CRUD zones/tournées ») et pas requise par le critère d'acceptation. À couvrir dans une tâche dédiée si un besoin d'endpoint explicite apparaît.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 4 : tests via `APITestCase` + création ORM directe, sans `factory_boy`/`pytest-django`

**Contexte :** `CONVENTIONS.md` §Tests prescrit `pytest-django` et `factory_boy`, mais ni l'un ni l'autre n'est encore dans `requirements.txt` (non installés depuis les Tâches 1-2). Le critère d'acceptation de la Tâche 4 demande explicitement des tests « via le client de test DRF ».
**Décision :** tests écrits avec `rest_framework.test.APITestCase` (compatible `manage.py test`, pas besoin de pytest) et création directe des objets via l'ORM plutôt que des factories, pour rester dans le périmètre de la tâche sans ajouter de nouvelle dépendance non demandée.
**Statut :** ✅ Résolu — voir entrée « Correctif Tâche 4 : migration vers pytest-django + factory_boy » ci-dessous.

## [RÉSOLU] Correctif Tâche 4 : migration des tests vers pytest-django + factory_boy

**Contexte :** l'écart signalé dans l'entrée précédente (tests `api/tests/*.py` de la Tâche 4 écrits en `APITestCase` + création ORM directe, alors que `CONVENTIONS.md` §Tests impose `pytest-django` + `factory_boy`) a été corrigé par une tâche correctif dédiée, sur la même branche `feat/api-crud-zones-tournees-auth-jwt`.
**Fix :**
- Dépendances ajoutées à `requirements.txt` (versions figées via `pip freeze` dans le conteneur `backend`) : `pytest==9.1.1`, `pytest-django==4.14.0`, `factory_boy==3.3.3` (+ transitives `Faker`, `iniconfig`, `packaging`, `pluggy`, `Pygments`).
- Config pytest : `pytest.ini` à la racine du dépôt (`DJANGO_SETTINGS_MODULE = autombalit_backend.settings`) — pas de `pyproject.toml` créé, le projet n'en avait pas et utilise `requirements.txt` (cohérent avec le choix de la Tâche 1).
- Factories `factory_boy` centralisées dans `core/tests/factories.py` (app propriétaire des modèles `Zone`/`Tournee`/`Societe`), réutilisables par toute app consommatrice : `UserFactory`, `SocieteFactory`, `ZoneFactory`, `TourneeFactory`.
- Les trois fichiers de tests (`api/tests/test_auth.py`, `test_zones.py`, `test_tournees.py`) réécrits en fonctions pytest avec `@pytest.mark.django_db`, fixtures partagées dans `api/tests/conftest.py` (`api_client`, `user`, `authenticated_client`), plus aucun `APITestCase`/création ORM directe.
- `UserFactory.password` implémenté via un hook `@factory.post_generation` explicite (`set_password` + `save()` conditionnel, `skip_postgeneration_save = True`) plutôt que `factory.PostGenerationMethodCall`, pour éviter un warning de dépréciation `factory_boy` (`_after_postgeneration` va cesser de sauvegarder automatiquement après hooks post-génération dans une future version majeure) sans perdre la persistance du mot de passe.
**Vérification :** `docker compose exec backend pytest` → 10 tests passent, aucun warning ; image `backend` reconstruite à partir du `requirements.txt` mis à jour et re-testée à froid (pas seulement `pip install` à chaud dans le conteneur existant) ; `python manage.py check` toujours propre.
**Statut :** ✅ Résolu

## [CHOIX] Tâche 5 : properties GeoJSON Zone limitées à `nom`, matching par `nom` exact

**Contexte :** `SPEC.md` §3 ne prévoit pas de FK société sur `Zone` (contrairement à ce que suggérait l'énoncé de la Tâche 5, Étape 0, qui évoquait « nom, société associée »). Le modèle `Zone` existant (Tâche 2) n'a que `nom` et `polygone`.
**Alternatives évaluées :** ajouter un champ `code` (identifiant stable côté SIG) ou une FK `societe` à `Zone`, versus rester sur le schéma minimal déjà en place.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — properties GeoJSON requises = `nom` uniquement. Correspondance création/mise à jour d'une zone par `nom` exact (`Zone.objects.update_or_create(nom=..., defaults={"polygone": ...})`). Aucun changement de modèle `Zone`.
**Statut :** 🔵 Choix assumé — à revoir si l'équipe SIG a besoin de renommer une zone sans perdre son historique (le matching par nom casse dans ce cas).

## [CHOIX] Tâche 5 : géométrie Zone limitée à `Polygon` (pas de `MultiPolygon`)

**Contexte :** `Zone.polygone` est un `PolygonField` (Tâche 2). L'énoncé de la Tâche 5 demandait de confirmer `Polygon`/`MultiPolygon`.
**Décision :** confirmé par l'utilisateur — seul le type `Polygon` est accepté à l'upload ; tout `MultiPolygon` (ou autre type) est rejeté avec un message explicite identifiant la feature en cause. Aucun changement de modèle.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 5 : validation CRS — absence de `crs` acceptée (WGS84 implicite), rejet seulement si explicitement non-4326

**Contexte :** les contraintes de la tâche imposent le SRID 4326 et prévoient des tests « SRID absent/différent ». Le GeoJSON (RFC 7946) est nativement toujours en WGS84/EPSG:4326 ; le membre `crs` n'est qu'un vestige de l'ancienne spec GeoJSON (2008), optionnel.
**Décision :** un GeoJSON sans membre `crs` est accepté (SRID 4326 assigné explicitement par le service de validation, cohérent avec la RFC) ; un membre `crs` explicitement présent et pointant vers autre chose que WGS84/CRS84/EPSG:4326 est rejeté. Une géométrie topologiquement invalide (auto-intersection, etc., détectée via `GEOSGeometry.valid`) est également rejetée.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 5 : import all-or-nothing — validation complète (erreurs accumulées) puis transaction atomique

**Contexte :** stratégie d'import à confirmer en Étape 0 ; le prompt de tâche recommandait all-or-nothing par défaut sauf avis contraire (aucun avis contraire exprimé).
**Décision :** toutes les features sont validées avant toute écriture en base ; les erreurs de validation sont accumulées et retournées en un seul appel (pas de fail-fast sur la première erreur), pour permettre à l'équipe SIG de corriger tous les problèmes en une seule passe. La persistance des zones validées a lieu ensuite dans une unique transaction atomique (`geo_import/services/zone_import.py`) — un échec base de données annule tout l'import.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 5 : endpoint d'upload GeoJSON limité aux `Zone` (pas encore `Tournee`)

**Contexte :** `SPEC.md` §7 décrit `POST /api/admin/geojson/upload/` comme couvrant « tournée ou zone », mais le Constat de la Tâche 5 ne porte que sur les zones (« Les zones sont censées provenir de GeoJSON pré-converti par l'équipe SIG »).
**Décision :** l'endpoint `/api/admin/geojson/upload/` implémenté dans cette tâche ne traite que les `Zone` (`FeatureCollection` de `Polygon`). L'upload du tracé de `Tournee` (`LineString`/`MultiLineString`) n'est pas couvert par cette tâche — à traiter dans une tâche dédiée si besoin, sur ce même endpoint ou un endpoint séparé à trancher à ce moment-là.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 6 : extrait OSRM Sénégal+Gambie complet, profil `car.lua` par défaut

**Contexte :** les données OSRM avaient déjà été préparées manuellement avant la Tâche 6 (extrait Geofabrik + pipeline `osrm-extract`/`osrm-partition`/`osrm-customize` déjà exécuté, service `osrm` déjà fonctionnel dans `docker-compose.yml`), le quartier pilote n'étant toujours pas choisi (`TODO.md` Phase 0). Étape 0 de la tâche demandait de confirmer le périmètre de l'extrait et le profil de routing.
**Alternatives évaluées :** refaire un extrait réduit à la région de Dakar (service plus léger en RAM) vs conserver l'extrait Sénégal+Gambie complet déjà préparé.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — conserver l'extrait Sénégal+Gambie complet (reste valable quel que soit le quartier pilote retenu, évite de refaire tout le pipeline une fois le quartier connu) et le profil `car.lua` par défaut (fourni par l'image `osrm/osrm-backend`), avec l'approximation « camion ≈ voiture » documentée comme point ouvert à revalider en Phase 5 (test pilote réel) — voir `SPEC.md` §1/§9 et `GUIDE_DU_DEVELOPPEUR.md` §1.6.
**Ajout scripté :** `autombalit-backend/scripts/prepare_osrm_data.sh` scripte le téléchargement (idempotent) + le pipeline extract/partition/customize, en alternative aux commandes manuelles déjà documentées dans `GUIDE_DU_DEVELOPPEUR.md` §1.6. Les fichiers `.osrm*` restent non commités (déjà exclus par `.gitignore` depuis la Tâche 1).
**Vérification :** `curl "http://localhost:5000/route/v1/driving/-17.4467,14.6928;-17.44,14.70"` retourne une route valide (`"code":"Ok"`) via le service `osrm` du `docker-compose.yml` existant, sans modification nécessaire de ce service (déjà correctement configuré depuis la Tâche 1).
**Statut :** 🔵 Choix assumé — extrait/profil à réévaluer une fois le quartier pilote choisi et testé en conditions réelles (Phase 5).

## [RÉSOLU] Tâche 7 : healthchecks Docker Compose et `depends_on: condition: service_healthy`

**Contexte :** un premier `docker compose up` (sans healthcheck) a démarré tous les services avec succès sur la machine de dev, mais sans garantie d'ordre réel de disponibilité (`depends_on` par ordre de démarrage seulement) — race condition potentielle sur une machine plus lente ou un premier démarrage PostGIS plus long.
**Décision :** ajout d'un `healthcheck` sur `db`, `redis`, `mosquitto`, `osrm`, et `depends_on: condition: service_healthy` sur `backend`/`mqtt_listener` pointant vers ces quatre services. Commandes de healthcheck choisies après inspection réelle des outils disponibles dans chaque image (pas d'hypothèse a priori) :
- `db` (postgis/postgis) : `pg_isready -U autombalit_user -d autombalit_dev` (outil standard déjà présent).
- `redis` (redis:7-alpine) : `redis-cli ping` (outil standard déjà présent).
- `mosquitto` (eclipse-mosquitto:2) : pas de `curl`/`wget`, mais `nc` présent → `nc -z -w3 localhost 1883`.
- `osrm` (osrm/osrm-backend, Debian stretch) : ni `curl`/`wget`/`nc`/`python3`, seul `bash` disponible → test de connexion `bash -c "timeout 3 bash -c '</dev/tcp/127.0.0.1/5000'"`.
- `backend` (image applicative Python) : pas de `curl`/`wget`/`nc`, `python3` disponible → `python -c "import socket; socket.create_connection(('localhost', 8000), timeout=3)"`. Pas d'endpoint `/health/` créé dans cette tâche (hors périmètre).
**Vérification :** chaque healthcheck testé manuellement en isolation (`docker compose exec <service> <commande>`) avant d'être déclaré dans `docker-compose.yml`, puis `docker compose up -d` complet confirmant la séquence réelle `Waiting → Healthy` pour `db`/`redis`/`mosquitto`/`osrm` avant le démarrage de `backend`/`mqtt_listener`, et `docker compose ps` confirmant `(healthy)` sur les cinq services fonctionnels (tout sauf `mqtt_listener`, dont l'échec attendu — Tâche 9 — est indépendant des healthchecks).
**Effet de bord corrigé :** ajout de `ENV PYTHONUNBUFFERED=1` dans le `Dockerfile` — sans cette variable, les logs Django n'apparaissaient pas en temps réel dans `docker compose logs` (bufferisés faute de TTY dans le conteneur), ce qui aurait rendu le futur débogage du `mqtt_listener` (Tâche 9) plus difficile.
**Statut :** ✅ Résolu

## [CHOIX] Tâche 7 : câblage Django ↔ Redis (Channels) volontairement non fait

**Contexte :** `redis`, `channels`, `channels_redis` sont dans `requirements.txt` depuis la Tâche 1, et le service `redis` est bien accessible en réseau depuis `backend` (`redis.Redis.from_url(...).ping()` → `True`, vérifié en conteneur), mais `settings.py` ne déclare ni `CHANNEL_LAYERS` ni `ASGI_APPLICATION`.
**Décision :** ne pas câbler Channels dans cette tâche — le périmètre de la Tâche 7 est l'orchestration Docker Compose (healthchecks, ordre de démarrage), pas l'implémentation applicative du temps réel citoyen (prévue en Phase 2, `SPEC.md` §6). "Redis accessible" (validé ici) est distinct de "Redis utilisé par Django" (à faire plus tard).
**Statut :** 🔵 Choix assumé — à lever lors de la tâche dédiée à `realtime`/Channels (Phase 2).

## [CHOIX] Tâche 8 : credentials MQTT liés au camion, pas au chauffeur

**Contexte :** `SPEC.md` §9 et l'énoncé initial de la Tâche 8 évoquaient une ACL « restreignant chaque chauffeur à son propre topic », mais le schéma de topic déjà fixé dans `CONVENTIONS.md` est `camions/{camion_id}/position`, et l'assignation chauffeur↔camion est journalière (`AssignationJournaliere`) — un chauffeur n'a donc pas de topic fixe à lui.
**Alternatives évaluées :** (A) credentials par camion, ACL statique ; (B) credentials par chauffeur avec ACL dynamique (plugin tiers type `mosquitto-go-auth` interrogeant `AssignationJournaliere` du jour) ; (C) credentials par chauffeur avec un topic redéfini par chauffeur (`camions/chauffeur/{chauffeur_id}/position`), nécessitant de dévier du nommage déjà établi.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — option A. Un compte MQTT par camion (`camion_<camion_id>`), autorisé à publier uniquement sur `camions/<camion_id>/position`, jamais en lecture. Le compte backend (`autombalit_backend`) garde un accès en lecture seule à `camions/#`. Reste dans la stack Mosquitto native (fichiers `passwd`/`acl`), pas de plugin d'auth tiers.
**Point ouvert :** cette tâche ne couvre que le provisioning des comptes camion côté Mosquitto (scripts `scripts/provision_mqtt_camion.sh`/`provision_mqtt_backend.sh`, `docker/mosquitto/passwd`/`acl` non commités). La façon dont le chauffeur assigné à un camion pour la journée récupère les credentials MQTT de ce camion (ex. réponse enrichie de `POST /api/tournees/du-jour/`) reste à concevoir dans une tâche ultérieure — probablement au moment de câbler l'app Flutter chauffeur (Phase 3) ou l'auth Firebase (Tâche 12).
**Vérification :** `scripts/test_mqtt_acl.sh` provisionne deux camions de test et confirme, via `mosquitto_pub`/`mosquitto_sub` réels sur le service `mosquitto` du `docker-compose.yml`, qu'un camion publie bien sur son propre topic et que sa tentative sur le topic d'un autre camion n'est jamais relayée ; vérifié aussi qu'un camion n'a aucun droit de lecture, pas même sur son propre topic (pas nécessaire, il ne fait que publier).
**Effet de bord corrigé :** l'image `eclipse-mosquitto:2` termine son process sur `SIGHUP` au lieu de recharger `passwd`/`acl` à chaud (testé), repris seulement grâce à `restart: unless-stopped` — les scripts utilisent donc `docker compose restart mosquitto`, plus explicite. Par ailleurs, `chmod 600`/`700` sur les fichiers `passwd`/`acl` montés depuis l'hôte casse la lecture côté conteneur (UID hôte ≠ UID `mosquitto` du conteneur) et fait planter Mosquitto au redémarrage suivant — permissions par défaut (world-readable, avertissement Mosquitto non bloquant) conservées volontairement en dev local.
**Statut :** 🔵 Choix assumé — point ouvert sur la distribution des credentials au chauffeur du jour à lever ultérieurement.

## [CHOIX] Tâche 9 : architecture du listener MQTT (management command + client/service/selector dédiés)

**Contexte :** `docker-compose.yml` (Tâche 1) attend déjà une commande `python manage.py run_mqtt_listener` dans le service `mqtt_listener`, en échec depuis la Tâche 7 faute d'implémentation (`Unknown command`). Étape 0 de la Tâche 9 demandait de confirmer ce mécanisme (process séparé) plutôt qu'un consumer Django Channels.
**Décision :** confirmé — process séparé, implémenté en couches (`CONVENTIONS.md`) :
- `tracking/clients/mqtt_client.py` (`PositionMQTTClient`) : encapsule `paho-mqtt` (callback API v2), s'abonne à `camions/+/position`, extrait le `camion_id` du topic, délègue à un callback injecté — aucune logique métier, avale toute exception du callback pour ne jamais crasher le listener sur un message individuel.
- `tracking/services/position_ingestion.py` (`ingest_position`) : parsing/validation du payload JSON (`lat`, `lng`, `horodatage` ISO 8601 timezone-aware), résolution du camion, vérification de la tournée active (voir entrées ci-dessous), création du `PositionCamion`. Payload malformé ou camion inconnu → log + `None` retourné, jamais d'exception remontée au client MQTT.
- `tracking/selectors/assignations.py` (`get_assignation_en_cours`) : lecture dédiée de l'`AssignationJournaliere` active.
- `tracking/management/commands/run_mqtt_listener.py` : commande Django minimale, cable juste le client au service.
**Vérification :** 17 tests (`pytest`, `factory_boy`) sur le service et le client (payload valide/malformé, camion inconnu, tournée non active, rafale) + validation de bout en bout avec le vrai stack Docker Compose (`mosquitto_pub` réel sur un compte `camion_<id>` provisionné, `mqtt_listener` réellement démarré) : position créée si tournée `en_cours`, rejetée sans écriture si `terminee`, payload malformé ignoré sans crash du listener (message valide suivant toujours traité). Ferme au passage l'avertissement de la Tâche 7 sur `mqtt_listener` qui bouclait en erreur.
**Statut :** ✅ Résolu

## [CHOIX] Tâche 9 : `camion_id` du topic MQTT = `Camion.pk`

**Contexte :** le topic `camions/{camion_id}/position` (Tâche 8) et son provisioning (`scripts/provision_mqtt_camion.sh <camion_id>`) traitaient `camion_id` comme un identifiant libre, sans lien formalisé avec un champ précis du modèle `Camion`.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — `camion_id` correspond à `Camion.pk` (id numérique). Le listener résout le camion via `Camion.objects.get(pk=camion_id)` ; un id inconnu ou non numérique est traité comme un camion inconnu (log + rejet, pas d'exception).
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 9 : vérification de tournée active par la date de l'horodatage du message

**Contexte :** la règle métier (`SPEC.md` §4.1, Tâche 9) impose de rejeter toute position hors tournée active. Il fallait trancher quelle date utiliser pour chercher l'`AssignationJournaliere (camion, date, statut=en_cours)` correspondante : la date portée par le message (`horodatage`) ou la date du jour au moment du traitement par le listener.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — date extraite de l'`horodatage` du message, pas `now()` au moment du traitement. Une position appartient à la journée qu'elle décrit ; ce choix traite correctement une rafale envoyée après minuit avec des timestamps de la veille (cf. règle métier sur les rafales, `SPEC.md` §4.6).
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 10 : facteur de correction ETA = vitesse effective par zone (km/h), moyenne mobile, seuil 5 passages

**Contexte :** `SPEC.md` §4.4 impose une valeur par défaut (~10 km/h) tant que l'historique est insuffisant (5-10 passages), sans préciser la formule exacte une fois l'historique suffisant, ni où la stocker — l'historique brut `PositionCamion` est purgé après 24-48h (`SPEC.md` §3), donc le facteur doit vivre dans un modèle persistant séparé plutôt que d'être recalculé depuis les positions brutes à chaque appel.
**Alternatives évaluées :** (A) vitesse effective par zone en km/h, ETA = distance OSRM / vitesse effective ; (B) multiplicateur appliqué à la durée OSRM brute (facteur dérivé, moins lisible car le défaut SPEC.md est exprimé en km/h, pas en ratio).
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — option A. Modèle `tracking.FacteurCorrectionZone` (`zone` OneToOne, `vitesse_effective_kmh` défaut 10.0, `nombre_observations`). `duree_corrigee_secondes = (distance_osrm_metres / 1000) / vitesse_effective_kmh * 3600`. Tant que `nombre_observations < 5` (seuil confirmé, borne basse de la fourchette `SPEC.md`), la vitesse par défaut (10 km/h) est utilisée à la place de la moyenne stockée, même partielle. Au-delà, mise à jour par moyenne mobile simple : `nouvelle_moyenne = (ancienne_moyenne * n + vitesse_observee) / (n + 1)`, via `tracking/services/facteur_correction.py::enregistrer_observation_zone` (verrouillage `select_for_update` pour éviter une race entre deux mises à jour concurrentes).
**Point ouvert :** `enregistrer_observation_zone` est implémentée et testée mais n'est appelée par aucun flux automatique à ce stade — détecter qu'un camion a terminé un passage dans une zone et en mesurer la vitesse réelle observée reste à concevoir (Phase 5, `TODO.md` : « Ajustement du facteur de correction par zone à partir des données terrain »), qui devra appeler cette fonction plutôt que d'en réinventer une.
**Implémentation :** `tracking/clients/osrm_client.py` (`OSRMClient.calculer_itineraire`, encapsule l'appel HTTP `route/v1/driving`, lève `OSRMError` sur timeout/connexion/HTTP/JSON/route introuvable) ; `tracking/services/eta.py` (`calculer_eta`, combine OSRM + vitesse effective, retourne `None` — pas d'exception ni de 500 — si OSRM est indisponible) ; `tracking/selectors/facteur_correction.py` (lecture de la vitesse à utiliser).
**Vérification :** tests unitaires (mocks `requests`/client OSRM, pas d'appel réseau réel) sur le client, le service ETA et la moyenne mobile ; vérification manuelle de bout en bout contre le vrai service `osrm` du `docker-compose.yml` (route réelle Dakar, calcul ETA avec vitesse par défaut).
**Statut :** 🔵 Choix assumé — formule à valider avec des données terrain réelles en Phase 5.

## [CHOIX] Tâche 11 : modèle anti-spam `SeuilNotifie` (camion + date séparés, pas de FK assignation)

**Contexte :** Étape 0 de la Tâche 11 demandait de confirmer le modèle de suivi anti-spam des notifications de seuil, deux options possibles : FK `AssignationJournaliere` (encapsule déjà camion+date) vs champs `camion`/`date` séparés.
**Alternatives évaluées :** (A) `camion` (FK) + `date` séparés, contrainte unique `(utilisateur, camion, date, seuil_minutes)` ; (B) FK `assignation`, contrainte unique `(utilisateur, assignation, seuil_minutes)`, plus DRY mais couplée au cycle de vie de l'assignation.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — option A. `notifications.SeuilNotifie (utilisateur FK, camion FK, date, seuil_minutes, notifie_le)`, contrainte unique sur les quatre premiers champs. Reste valable même si l'`AssignationJournaliere` du jour est modifiée après coup.
**Statut :** 🔵 Choix assumé

## [CHOIX] Tâche 11 : logique de seuils dans `notifications`, câblée depuis le listener MQTT

**Contexte :** le calcul ETA (Tâche 10) existe, mais aucun déclenchement de notification. `SPEC.md` §5 place la « logique de seuils » dans l'app `notifications`, cohérent avec le modèle `SeuilNotifie` ci-dessus.
**Décision :**
- `tracking/selectors/positions.py::est_la_plus_recente` : vérifie qu'aucune position plus récente n'existe déjà pour le camion (SPEC.md §4.6) — seule la plus récente d'une rafale déclenche le calcul, les autres restent persistées pour l'historique (déjà fait Tâche 9).
- `notifications/selectors/utilisateurs_concernes.py::get_points_enregistres_dans_zones` : lecture des `PointEnregistre` dont la zone est couverte par la tournée de l'assignation active.
- `notifications/services/seuils.py::evaluer_seuils_pour_position` : pour chaque utilisateur concerné, calcule l'ETA (réutilise `tracking.services.eta.calculer_eta`, dégradation propre si OSRM indisponible) et crée un `SeuilNotifie` (via `get_or_create`, course géré par la contrainte unique + gestion Django de l'`IntegrityError`, SPEC.md §13) pour chaque seuil ≤ ETA courant non encore notifié aujourd'hui. Si plusieurs seuils sont franchis d'un coup (ETA ayant chuté brutalement entre deux positions), tous les seuils non notifiés sont déclenchés en un seul appel plutôt qu'un seul à la fois — pas de suivi de l'ETA précédente, uniquement de l'état « déjà notifié » par seuil.
- `notifications/services/dispatch.py::notifier_seuil_franchi` : point d'extension explicite vers l'envoi FCM (Tâche 13), injecté en paramètre optionnel de `evaluer_seuils_pour_position` (même pattern DI que `osrm_client` dans `calculer_eta`) — se limite à un `logger.info` pour l'instant.
- Câblage : `tracking/management/commands/run_mqtt_listener.py` appelle désormais `ingest_position` puis, si une position a été créée, `evaluer_seuils_pour_position` — toujours une commande minimale, aucune logique métier propre (celle-ci reste dans les services dédiés, conforme à `CONVENTIONS.md`).
**Vérification :** 9 nouveaux tests (`notifications/tests/test_seuils.py`, `tracking/tests/test_positions_selector.py`), mocks sur `calculer_eta` (jamais d'appel OSRM réel en test, conforme à `CONVENTIONS.md` §Tests) : franchissement de seuil unique, franchissement simultané de plusieurs seuils, anti-spam (pas de doublon le même jour), position non-la-plus-récente d'une rafale ignorée (OSRM jamais appelé dans ce cas), ETA indisponible → aucune notification. Suite complète : 59 tests passent.
**Statut :** ✅ Résolu

## [CHOIX] Tâche 12 : Chauffeur pré-créé par un admin (Django admin), pas d'auto-inscription

**Contexte :** Étape 0 de la Tâche 12 demandait de confirmer le flux d'échange token Firebase → JWT ; restait ouvert comment un nouveau Chauffeur (avec sa `societe`, FK obligatoire) apparaît côté backend avant sa première connexion, aucune interface d'auto-inscription n'existant.
**Alternatives évaluées :** (A) Chauffeur pré-créé par un admin société (`core.admin`, déjà enregistré) — l'échange Firebase ne fait que retrouver le Chauffeur par téléphone (claim du token) et lui attacher son `firebase_uid` au premier login réussi ; (B) auto-inscription du chauffeur à la première connexion, avec un `societe_id` fourni dans le payload d'échange (suppose une sélection de société côté app, non construite).
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — option A. Un Chauffeur sans `firebase_uid` encore attaché est identifiable par téléphone. Aucun Chauffeur trouvé pour le téléphone du token Firebase → erreur explicite (403, "contactez votre société de collecte"), pas de création automatique.
**Effet de bord modèle :** `Chauffeur.firebase_uid` passé de `unique=True` (obligatoire) à `unique=True, null=True, blank=True` — plusieurs chauffeurs pré-créés sans `firebase_uid` doivent pouvoir coexister (NULL ≠ NULL pour la contrainte unique en PostgreSQL). Ajout d'un champ `user` (OneToOneField vers `settings.AUTH_USER_MODEL`, nullable) sur `Chauffeur` et `Utilisateur` : un compte technique `django.contrib.auth.User` (sans mot de passe utilisable, seule l'authentification Firebase permet d'en obtenir un JWT) porte le JWT applicatif et est lié 1-1 au modèle métier — cohérent avec le pattern déjà en place depuis la Tâche 4 (`UserFactory`/`force_authenticate` dans les tests DRF), sans introduire de `AUTH_USER_MODEL` personnalisé.
**Statut :** 🔵 Choix assumé — la distribution des identifiants de connexion au chauffeur pré-créé (comment il apprend qu'un compte existe pour son téléphone) reste hors périmètre, à traiter avec l'écran de validation manuelle (Phase 4).

## [CHOIX] Tâche 12 : ancien endpoint `/api/token/` (username/password, Tâche 4) supprimé

**Contexte :** `/api/token/` avait été ajouté en Tâche 4 comme mécanisme JWT provisoire (username/password, `TokenObtainPairView`) en attendant l'auth Firebase réelle, explicitement documenté comme tel dans `DECISIONS.md`.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — supprimé (`autombalit_backend/urls.py`, `api/tests/test_auth.py`), remplacé par `/api/auth/token/` (échange Firebase → JWT) et `/api/auth/token/refresh/` (inchangé, `TokenRefreshView` de `simplejwt`), conformes à `SPEC.md` §7. Les deux routes JWT vivent désormais dans `api/urls.py` (avec le reste du routage DRF) plutôt qu'au niveau racine du projet.
**Statut :** ✅ Résolu

## [CHOIX] Tâche 12 : architecture de l'échange token Firebase → JWT (client/services dédiés)

**Contexte :** SPEC.md §7 définit `POST /api/auth/token/` sans détailler l'implémentation ; CONVENTIONS.md impose un client dédié pour toute intégration externe (ici Firebase Admin) et un service layer pour la logique métier.
**Décision :**
- `api/clients/firebase_client.py` (`verifier_id_token`) : encapsule `firebase_admin.auth.verify_id_token`, initialise le SDK Firebase Admin en lazy (jamais au chargement du module, jamais en test — voir `_sans_vrai_sdk_firebase` dans les tests), lève `FirebaseTokenError` si le token est invalide/expiré/révoqué ou sans `phone_number`.
- `core/selectors/chauffeurs.py::get_chauffeur_par_telephone` + `core/services/chauffeurs.py::attacher_firebase_uid` (lève `ChauffeurInconnuError`/`FirebaseUidConflitError`, `core/services/exceptions.py`).
- `citizens/services/utilisateurs.py::get_or_create_utilisateur_firebase` (auto-inscription citoyen, lève `TelephoneDejaUtiliseError` si le téléphone est déjà associé à un autre `firebase_uid`).
- `api/services/auth_exchange.py::echanger_token_firebase` : orchestre les trois ci-dessus selon le `role` (`citoyen`/`chauffeur`) et retrouve/crée le `django.contrib.auth.User` technique lié.
- `api/permissions.py::EstChauffeurValide` : permission DRF réutilisable pour restreindre un futur endpoint aux chauffeurs `statut_validation == valide` — aucun endpoint chauffeur réel n'existe encore (Phase 3), donc testée directement (`api/tests/test_permissions.py`) plutôt que câblée sur une vue.
- `api/views.py::FirebaseTokenExchangeView` : `AllowAny`, traduit les exceptions métier en réponses DRF explicites (`AuthenticationFailed` 401 pour un token Firebase invalide, `PermissionDenied` 403 pour un chauffeur inconnu/conflit d'uid, `ValidationError` 400 pour un `role` absent/invalide via le serializer).
**Vérification :** 79 tests passent (`docker compose exec backend pytest`), `python manage.py check`/`migrate` propres. Mocks systématiques du vérificateur Firebase (jamais de vrai appel SDK/réseau en test, conforme à `CONVENTIONS.md` §Tests).
**Statut :** ✅ Résolu

## [CHOIX] Tâche 13 : intégration FCM — token existant réutilisé, nettoyage sur token invalide

**Contexte :** Étape 0 de la Tâche 13 demandait de confirmer l'emplacement de stockage des tokens FCM des appareils citoyens.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — `Utilisateur.fcm_token` (déjà posé en Tâche 2) réutilisé tel quel, aucune migration nécessaire. Sur un token invalide/non enregistré signalé par FCM (`messaging.UnregisteredError`), `Utilisateur.fcm_token` est vidé (`''`) plutôt que marqué via un champ de statut séparé — plus simple, cohérent avec le champ existant ; l'utilisateur redevient joignable dès que l'app en renvoie un nouveau.
**Architecture :** `notifications/clients/fcm_client.py` (`envoyer_notification`, `FCMError`/`FCMTokenInvalideError`) encapsule `firebase_admin.messaging`, avec sa propre app Firebase lazy (`_get_app`) qui réutilise l'app par défaut déjà initialisée par `api/clients/firebase_client.py` si elle existe dans le même process (`firebase_admin.get_app()`), sinon l'initialise elle-même — un seul SDK Firebase Admin par process quel que soit l'ordre d'appel entre auth et notifications. `notifications/services/dispatch.py::notifier_seuil_franchi` (point d'extension préparé Tâche 11) appelle ce client : aucun token → envoi ignoré (log) ; `FCMTokenInvalideError` → nettoyage du token ; `FCMError` générique (réseau, quota...) → log, jamais d'exception remontée (le traitement de la position ne doit jamais échouer à cause d'un envoi FCM raté, SPEC.md §13).
**Point de vigilance :** `firebase_admin` 7.5.0 déprécie `messaging.Message(token=...)` au profit de `fid=` (Firebase Installation ID) — `token=` conservé volontairement ici car c'est ce que produit `FirebaseMessaging.instance.getToken()` côté client Flutter (cohérent avec le nom `Utilisateur.fcm_token`) ; génère un `DeprecationWarning` non bloquant dans les tests, à surveiller si une future version du SDK retire complètement `token`.
**Vérification :** `docker compose exec backend pytest` → 86 tests passent (mocks systématiques sur `firebase_admin.messaging.send`, aucun appel réseau réel, conforme à `CONVENTIONS.md` §Tests) ; `python manage.py check` et `makemigrations --check --dry-run` propres (aucune migration générée, confirmant l'absence de changement de modèle).
**Statut :** ✅ Résolu

## [RÉSOLU] Complément backend Tâche 14 : `/api/tournees/du-jour/`, `/start/`, `/stop/` + statut chauffeur dans l'auth

**Contexte :** l'Étape 0 de la Tâche 14 (`autombalit-mobile`) supposait déjà existants `POST /api/tournees/du-jour/`, `POST /api/tournees/{id}/start/` et `/stop/` (SPEC.md §7), ainsi qu'un moyen de connaître le `statut_validation` du chauffeur après connexion. Vérification du code réel (`api/urls.py`, `api/views.py`) avant de coder côté Flutter : aucun des trois endpoints n'existait (seul un CRUD générique `/api/tournees/`), et `FirebaseTokenExchangeView` ne renvoyait que `{access, refresh}`. Signalé à l'utilisateur avant de coder (pattern déjà anticipé par la Tâche 15 pour un cas similaire) ; décision : compléter le backend d'abord, dans une tâche ad-hoc non numérotée dans `TASK_PROMPTS.md`.
**Implémentation (dépôt `autombalit-backend`, branche `feat/tournees-du-jour-start-stop`, au-dessus de `feat/fcm-notifications`) :**
- `tracking/selectors/assignations.py::get_assignation_du_jour_chauffeur(chauffeur, date)` : lecture de l'`AssignationJournaliere` du chauffeur pour une date, quel que soit son statut.
- `tracking/services/assignations.py::demarrer_tournee`/`arreter_tournee` : transitions `planifiee→en_cours` et `en_cours→terminee` uniquement ; toute autre transition lève `TransitionAssignationInvalideError` (`tracking/services/exceptions.py`), traduite en 400 côté API.
- `api/serializers.py::AssignationJournaliereSerializer` (+ `CamionSerializer`) : expose `id, date, statut, camion{id,immatriculation}, tournee` (réutilise `TourneeSerializer` existant, donc `zones` en liste d'ids — cohérent avec le choix déjà pris Tâche 4).
- Trois actions DRF (`@action`) sur le `TourneeViewSet` existant plutôt que des vues/routes séparées : `du_jour` (`detail=False`, `url_path='du-jour'`), `start`/`stop` (`detail=True`, donc `pk` = id de la `Tournee`, conforme à `SPEC.md §7`). Ce choix évite tout risque d'ordre de routes avec le CRUD déjà enregistré par le `DefaultRouter` (Tâche 4) — DRF place les routes `detail=False` avant la route détail générique.
- Permission dédiée sur ces trois actions (`get_permissions` override) : `EstChauffeurValide` (déjà créée Tâche 12, jusqu'ici jamais câblée sur une vue réelle) — un chauffeur `en_attente`/`rejete` reçoit 403, conforme à SPEC.md §4.3.
- `FirebaseTokenExchangeView` : ajoute `statut_validation` dans la réponse JSON quand un `Chauffeur` est résolu (absent pour le rôle `citoyen`) — évite un appel supplémentaire pour que l'app sache immédiatement si elle doit afficher l'écran d'attente.
**Vérification :** 17 nouveaux tests (103 au total), `python manage.py check` et `makemigrations --check --dry-run` propres (aucun changement de modèle).
**Statut :** ✅ Résolu (implémenté, non commité au moment de la rédaction — en attente d'accord explicite avant commit, comme toujours).

## [CHOIX] Tâche 14 : architecture Flutter de l'auth OTP chauffeur + tournée du jour

**Contexte :** premier câblage réel de Firebase/réseau côté mobile (Tâche 3 n'était que du scaffolding). Plusieurs choix d'implémentation non explicitement tranchés par `SPEC.md`/`CONVENTIONS.md` au niveau du détail.
**Décisions :**
- **Packages ajoutés** (`pubspec.yaml`, versions figées explicitement — leçon de la Tâche 3 sur les versions non figées) : `firebase_core`/`firebase_auth` (OTP), `flutter_secure_storage` (JWT, imposé par `CONVENTIONS.md`), `http` (appels API, plus léger que `dio` pour ce besoin), `provider` (state management — correspond au terme « providers » déjà utilisé dans `CONVENTIONS.md §Architecture`) ; `mocktail` en dev pour les tests widget avec mocks.
- **Session de référence = JWT applicatif, pas la session Firebase** : `FirebaseOtpService.confirmerCode` appelle `signOut()` juste après avoir récupéré l'ID token, pour éviter deux sources de vérité sur l'état « connecté » (Firebase persiste sa session par défaut, ce qui court-circuiterait l'OTP au prochain lancement sans que l'app le sache).
- **Couche `services/` vs `repositories/` côté Flutter** : `lib/common/services/` = appels externes bruts (Firebase SDK, HTTP — équivalent des `clients/` backend) ; `lib/driver/repositories/` = composition orientée domaine consommée par les providers (`AuthRepository`, `TourneeRepository`), y compris quand la donnée est distante sans cache local pour l'instant (contrairement à la définition littérale de `CONVENTIONS.md` qui limite `repositories/` au local/cache) — interprétation à vérifier/ajuster si besoin lors de la Tâche 16 (queue hors-ligne) ou 19 (cache citoyen), qui elles impliquent clairement du stockage local dans cette même couche.
- **`ApiClient` (`lib/common/services/api_client.dart`)** : déballe l'enveloppe standard `{success,data,error}` (`CONVENTIONS.md`), tente un rafraîchissement du JWT + une seule relance sur 401, abandonne la session (`SessionExpiredException`) si le refresh échoue aussi — pas de gestion de rotation de refresh token (non activée côté backend, `SIMPLE_JWT` par défaut).
- **Affichage des zones de la tournée** : seul le nombre de zones est affiché (`TourneeSerializer.zones` ne renvoie que des ids depuis la Tâche 4, pas de noms) — pas de changement de serializer au-delà de ce qui était nécessaire pour Tâche 14, pour ne pas élargir le périmètre du complément backend ci-dessus.
**Non fait dans cette tâche** (hors périmètre annoncé) : capture GPS/MQTT (Tâche 15), queue hors-ligne (Tâche 16), thème visuel complet au-delà de la couleur de fond/seed ambre (SPEC.md §10 — reste à affiner visuellement).
**Vérification :** `flutter analyze` propre à l'exception de `lib/main_chauffeur.dart` (import de `firebase_options.dart`, généré par `flutterfire configure` — étape manuelle non faite par Claude Code, voir `GUIDE_DU_DEVELOPPEUR.md` §2.4) ; `flutter test` (7 tests, dont 6 nouveaux sur les écrans OTP et tournée du jour, mocks sur les repositories) tous verts ; `flutter build apk --flavor chauffeur --debug` va jusqu'à la compilation Dart (confirme qu'aucune dépendance ajoutée ne casse la config Gradle/minSdk) et échoue uniquement sur l'import `firebase_options.dart` manquant, comme attendu.
**Suite (session de test sur appareil physique) :** voir l'entrée « Test sur appareil physique + correctif vérification automatique OTP » ci-dessous — `flutterfire configure` exécuté par l'utilisateur, test réel effectué, un bug corrigé.
**Statut :** 🔵 Choix assumé

## [RÉSOLU] Tâche 14 : test sur appareil physique + correctif vérification automatique OTP

**Contexte :** suite de l'entrée précédente, une fois `flutterfire configure` exécuté par l'utilisateur (package `com.autombalit.chauffeur`, `google-services.json`/`lib/firebase_options.dart`/`firebase.json` générés) et le Samsung physique branché (`adb devices` → `RFCRA1E10HH`).
**Problème 1 rencontré :** `CONFIGURATION_NOT_FOUND` de Firebase Auth au premier essai. **Cause :** SHA-256 du certificat de debug non enregistrée dans la console Firebase pour l'app Android `com.autombalit.chauffeur` (seul le SHA-1 avait été ajouté). **Fix :** utilisateur a ajouté le SHA-256 (`88:7C:40:75:...`) dans Firebase Console → Paramètres du projet → l'app Android. Résolu après réinstallation/relance de l'app (pas besoin de rebuild, changement uniquement côté console).
**Problème 2 rencontré (bug réel, corrigé dans le code) :** sur l'appareil qui possède la carte SIM du numéro testé, Android déclenche la vérification automatique de Firebase (SMS Retriever / vérification instantanée) — `verificationCompleted` se déclenche avec un credential déjà valide, sans jamais passer par `codeSent`. Le code initial de `FirebaseOtpService.envoyerCode` ignorait ce credential (`verificationCompleted: (_) {}`), laissant l'app bloquée sur l'écran de saisie du téléphone alors que Firebase avait déjà validé en interne — l'utilisateur ne voyait jamais l'écran de saisie du code alors qu'un SMS avait bien été envoyé en parallèle.
**Fix :** `FirebaseOtpService` gère maintenant les deux chemins (`verificationCompleted` et code saisi manuellement) via une méthode privée commune (`_connecterEtRecupererIdToken`) ; `AuthRepository.envoyerCodeOtp` et `AuthProvider.envoyerCode` acceptent un nouveau callback `surConnexionAutomatique` qui connecte directement l'utilisateur (statut chauffeur inclus) sans passer par `EtapeAuth.saisieCode`. Nouveau test (`test/driver/auth_provider_test.dart`) couvrant ce chemin.
**Décision annexe :** `google-services.json`/`firebase_options.dart`/`firebase.json` committés tels quels (pas de `.gitignore`) — dépôt privé, valeurs non sensibles selon la documentation Firebase elle-même (sécurité assurée par les règles Firebase/App Check, pas par la confidentialité de l'`apiKey`), évite à chaque développeur de relancer `flutterfire configure`.
**Vérifié sur l'appareil physique (Samsung SCG10)** : build/install/lancement du flavor chauffeur réussis ; échange OTP Firebase → JWT applicatif confirmé de bout en bout contre le vrai backend (avec le numéro de test Firebase `+221771234567` : rejet 403 attendu tant qu'aucun `Chauffeur` n'existait pour ce numéro — voir entrée backend `seed_chauffeur_test` ci-après).
**Reste à vérifier manuellement dans l'UI** (non bloquant, couvert par les tests automatisés) : écran tournée du jour et actions start/stop sur l'appareil physique, une fois un `Camion`/`Tournee`/`AssignationJournaliere` du jour rattachés au chauffeur de test `+221771234567` (pas encore créés, en attente d'accord explicite sur les valeurs de test — voir aussi la commande `seed_chauffeur_test`).
**Statut :** ✅ Résolu (bug de vérification automatique) — test manuel complet encore partiel (OTP validé, tournée/start-stop en attente)

## [RÉSOLU] Backend : commande `seed_chauffeur_test` pour débloquer le test manuel Tâche 14

**Contexte :** le test manuel sur téléphone physique (entrée précédente) nécessitait un `Chauffeur` pré-créé et validé pour le numéro de test Firebase `+221771234567` — règle métier « pas d'auto-inscription » (DECISIONS.md Tâche 12) empêchant toute connexion sans ce préalable, confirmé par le rejet 403 observé.
**Décision :** commande de management Django reproductible plutôt qu'une manipulation one-shot en base (contrainte explicite du prompt) : `autombalit-backend/core/management/commands/seed_chauffeur_test.py` — crée/réutilise une `Societe` de test (« Société Test Pilote », déjà existante depuis un test manuel antérieur avec un autre numéro) et crée/met à jour un `Chauffeur` (`--telephone`, `--nom`, `--societe-nom`) avec `statut_validation=valide`. `firebase_uid` volontairement laissé vide : il s'attache automatiquement à la première connexion réussie (`core/services/chauffeurs.py::attacher_firebase_uid`, Tâche 12) — pas besoin d'aller le chercher dans la console Firebase au préalable.
**Exécuté pour** `+221771234567` (id=5, `Société Test Pilote`). Un autre chauffeur de test (`+221781531736`, id=4) avait déjà été créé manuellement (hors commande) lors d'un test antérieur dans la même société, avec `Camion`/`Tournee`/`AssignationJournaliere` du jour associés.
**Point ouvert signalé à l'utilisateur, pas encore tranché :** tester l'écran « tournée du jour » de façon significative pour le chauffeur `+221771234567` nécessite un `Camion`/`Tournee`/`AssignationJournaliere` du jour dédiés (ou réutiliser ceux du chauffeur `+221781531736`) — pas créés dans cette tâche, en attente d'accord explicite.
**Vérification :** commande testée idempotente (rejouée sans doublon), `python manage.py check` propre.
**Statut :** ✅ Résolu (commande) — voir suite ci-dessous (données de tournée créées)

## [RÉSOLU] Backend : commande `seed_tournee_test` — camion/zone/tournée/assignation pour start/stop

**Contexte :** suite de l'entrée précédente — le chauffeur de test `+221771234567` n'avait toujours aucune tournée assignée (écran « Aucune tournée aujourd'hui » confirmé correct sur l'appareil physique), nécessaire pour valider start/stop (Tâche 14) et préparer le test de la capture GPS/MQTT (Tâche 15, qui a aussi besoin d'une tournée `en_cours`).
**Décision :** deuxième commande de management reproductible, `core/management/commands/seed_tournee_test.py` (`--telephone-chauffeur` requis, `--immatriculation`/`--nom-zone`/`--nom-tournee`/`--nom-societe` avec défauts) — réutilise la Societe de test, crée/réutilise `Camion` (`TEST-001`), `Zone` (`Zone Test`, polygone factice sans contour géographique réel — pas nécessaire pour ce test), `Tournee` (`Tournée Test`) liée à la zone via `TourneeZone(ordre_passage=0)`, puis `update_or_create` l'`AssignationJournaliere` du jour (`camion`+`date` — respecte la contrainte unique existante) avec `statut=planifiee`. Échoue explicitement si le chauffeur n'existe pas encore (indique de lancer `seed_chauffeur_test` d'abord).
**Effet de bord assumé :** relancer la commande réinitialise le `statut` de l'assignation à `planifiee` à chaque exécution (pratique pour rejouer un test start/stop complet, mais écrase un statut `en_cours`/`terminee` en cours de test si on la relance par erreur).
**Documentation** : procédure complète de recréation du jeu de données de test (les deux commandes, dans l'ordre) consignée dans `autombalit-docs/FIXTURES_TEST.md` plutôt que dupliquée ici.
**Vérification :** commande testée idempotente (rejouée sans doublon, assignation réutilisée), `python manage.py check` propre.
**Statut :** ✅ Résolu — données prêtes pour le test manuel start/stop sur l'appareil physique

## [RÉSOLU] Complément backend Tâche 15 : distribution des credentials MQTT du camion assigné

**Contexte :** l'Étape 0 de la Tâche 15 (`autombalit-mobile`) devait trancher le point resté ouvert depuis la Tâche 8 : comment le chauffeur récupère les credentials MQTT du camion qui lui est assigné le jour même. Vérification du code réel avant de coder côté Flutter : ni `/api/tournees/du-jour/` ni `/start/` n'exposaient d'identifiant camion MQTT, et plus fondamentalement `scripts/provision_mqtt_camion.sh` n'a jamais stocké le mot de passe généré nulle part (affiché une seule fois, `docker/mosquitto/passwd` ne contient qu'un hash bcrypt) — le backend n'avait donc structurellement aucun moyen de le restituer. Signalé à l'utilisateur avant de coder (`AskUserQuestion`, pattern déjà anticipé par la Tâche 14 pour un cas similaire).
**Alternatives évaluées :** (A) stocker le mot de passe chiffré côté backend (nouveau champ + commande de provisioning dédiée), exposé via l'API au chauffeur assigné ; (B) contourner MQTT côté mobile — le chauffeur publierait sa position via un endpoint HTTPS authentifié JWT, le backend écrivant `PositionCamion` directement ou republiant en interne sur MQTT.
**Décision :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — option A, reste dans l'architecture déjà validée (Tâche 8 : compte statique par camion, stack Mosquitto native), sans revenir sur le choix MQTT chauffeur→backend (`SPEC.md`/`DECISIONS.md`).
**Implémentation (dépôt `autombalit-backend`, branche `feat/tournees-du-jour-start-stop`) :**
- `Camion.mqtt_password_chiffre` (`CharField`, vide par défaut) : mot de passe MQTT chiffré (Fernet), jamais stocké en clair.
- `MQTT_CREDENTIALS_ENCRYPTION_KEY` (`settings.py`, nouvelle variable d'environnement, `.env.example` mis à jour) : clé Fernet de chiffrement, générée une fois (`Fernet.generate_key()`), jamais commitée (`.env`).
- `core/services/mqtt_credentials.py` : `nom_utilisateur_mqtt(camion)` (convention `camion_<id>`, déjà fixée par le script de provisioning), `chiffrer_mot_de_passe`/`dechiffrer_mot_de_passe` (Fernet ; `dechiffrer_mot_de_passe` renvoie `None` sur token vide ou illisible plutôt que de lever une exception — un camion pas encore provisionné ne doit jamais faire échouer l'appelant).
- `core/management/commands/set_mqtt_password_camion.py` : enregistre (chiffré) le mot de passe d'un camion déjà provisionné côté Mosquitto — appelée automatiquement par `scripts/provision_mqtt_camion.sh` juste après le provisioning, avec le même mot de passe (pas de nouvelle génération, une seule source de vérité pour le mot de passe en clair au moment du provisioning).
- `AssignationJournaliereSerializer` : ajoute `mqtt_username`/`mqtt_password` (`SerializerMethodField`), donc exposés dans les réponses de `/api/tournees/du-jour/`, `/start/` et `/stop/`, au chauffeur validé assigné à ce camion aujourd'hui uniquement (permission `EstChauffeurValide` + résolution de l'assignation déjà scopée à `request.user.chauffeur`, Tâche 14). `mqtt_password` vaut `None` si le camion n'a pas encore de compte MQTT provisionné.
**Vérification :** 112 tests passent (`docker compose exec backend pytest`, dont 8 nouveaux — chiffrement/déchiffrement, commande de management, exposition des champs sur `du-jour`/`start`, absence si non provisionné) ; `python manage.py check`/`makemigrations --check --dry-run` propres. Provisioning réel du camion de test `TEST-001` (id=4, `scripts/provision_mqtt_camion.sh 4`) : compte Mosquitto créé, mot de passe accepté par un `mosquitto_pub` réel sur `camions/4/position`, et retrouvé identique en le déchiffrant depuis la base (`core.services.mqtt_credentials.dechiffrer_mot_de_passe`).
**Statut :** ✅ Résolu — implémenté, non commité au moment de la rédaction, en attente d'accord explicite avant commit (comme toujours).

## [RÉSOLU] Tâche 15 : capture GPS + publication MQTT côté app Chauffeur

**Contexte :** le start/stop tournée existait (Tâche 14) mais ne déclenchait encore aucune capture ni publication réelle. Le point ouvert de distribution des credentials MQTT a été traité séparément (voir l'entrée « Complément backend Tâche 15 » ci-dessus).
**Implémentation (dépôt `autombalit-mobile`, branche `feat/chauffeur-gps-mqtt`) :**
- Packages ajoutés (versions figées) : `geolocator: ^14.0.2` (capture GPS), `mqtt_client: ^10.11.11` (publication MQTT), `fake_async: ^1.3.3` en dev (tests du minuteur de publication adaptatif).
- `lib/driver/services/position_stream_service.dart` : encapsule `geolocator` (demande de permission runtime, flux de positions). `lib/driver/services/mqtt_position_client.dart` : encapsule `mqtt_client` (connexion authentifiée, publication QoS 1 sur `camions/{camion_id}/position`, déconnexion) — équivalents des `clients/` backend (CONVENTIONS.md §Architecture).
- `lib/driver/repositories/tracking_repository.dart` : orchestre les deux services. `demarrer`/`arreter` idempotents. Cycle de vie strictement lié à la tournée : jamais de capture hors tournée active (SPEC.md §4.1).
- `AssignationJournaliereSerializer`/`AssignationDuJour` (Flutter) exposent désormais `camion.id`, `mqtt_username`, `mqtt_password` — câblés depuis `TourneeProvider` (callbacks `onTourneeDemarree`/`onTourneeArretee`, pas de dépendance directe entre les deux `ChangeNotifier`) vers `TrackingProvider`, qui pilote `TrackingRepository`. `TourneeProvider.charger()` relance aussi le suivi si l'écran est rechargé alors qu'une tournée est déjà `en_cours` (redémarrage de l'app en cours de tournée).
- Indicateur d'état GPS actif (SPEC.md §11) sur `TourneeDuJourScreen` : bandeau vert si actif, bandeau d'erreur explicite si permission refusée / connexion MQTT impossible / camion non provisionné.
- Permissions Android runtime : `ACCESS_FINE_LOCATION`/`ACCESS_COARSE_LOCATION` ajoutées à `AndroidManifest.xml` (premier plan uniquement, voir point ouvert ci-dessous) ; message d'erreur clair dans l'UI si refusées, tournée non bloquée pour autant côté backend (juste pas de publication).
**Formule de fréquence adaptative** (TASK_PROMPTS.md, à documenter) : basée sur la vitesse instantanée du GPS (`Position.speed`, m/s) — camion en mouvement (≥ 2,0 m/s, ~7,2 km/h) → publication toutes les 10 s ; camion à l'arrêt (< 2,0 m/s) → toutes les 30 s. Seuil choisi pour rester sous la vitesse d'un pas rapide/démarrage tout en couvrant les arrêts fréquents de collecte (SPEC.md §1, tous les 30-50 m). Implémentée dans `TrackingRepository._planifierProchainePublication` via un `Timer` qui se reprogramme après chaque publication (plutôt qu'un `Timer.periodic` fixe) pour pouvoir changer d'intervalle d'un cycle à l'autre ; publie la dernière position connue (flux GPS continu, `distanceFilter: 0`) au lieu d'attendre un déplacement minimal, pour garantir un battement même à l'arrêt prolongé.
**Point ouvert assumé (non tranché explicitement avec l'utilisateur) :** capture GPS au premier plan uniquement (app ouverte) — pas de foreground service Android ni de permission `ACCESS_BACKGROUND_LOCATION`. Le tracking s'interrompt donc si l'app est mise en arrière-plan/l'écran verrouillé pendant la tournée. Décision prise pour rester dans le périmètre annoncé de la Tâche 15 (qui exclut déjà la robustesse hors-ligne, Tâche 16) ; à revisiter si le test pilote réel (Phase 5) montre que les chauffeurs verrouillent leur téléphone pendant la tournée.
**Non fait dans cette tâche** (hors périmètre annoncé) : file d'attente locale hors-ligne / retry MQTT (Tâche 16).
**Vérification :** `flutter analyze` propre, `flutter test` (18 tests, dont 8 nouveaux : formule d'intervalle adaptatif, cycle de vie `TrackingRepository` avec minuteur simulé via `fake_async`, gestion d'erreurs `TrackingProvider`). `flutter build apk --flavor chauffeur --debug` réussi (valide la compatibilité Gradle des nouvelles dépendances natives). **Test de bout en bout sur l'appareil physique** (Samsung, chauffeur de test `+221771234567`, camion `TEST-001` provisionné MQTT — voir complément backend) : démarrage de la tournée → 4 `PositionCamion` créées à intervalle exact de 30 s (14:50:56, 14:51:26, 14:51:56, 14:52:26 — camion à l'arrêt en intérieur, cohérent avec la formule), bandeau « GPS actif » affiché ; arrêt de la tournée → aucune nouvelle position sur les 45 s suivantes (poll vérifié), écran repasse à « Tournée terminée » sans indicateur GPS.
**Statut :** ✅ Résolu — implémenté et vérifié de bout en bout sur appareil physique, non commité au moment de la rédaction, en attente d'accord explicite avant commit (dépôts `autombalit-backend` et `autombalit-mobile` séparément, comme toujours).

## [RÉSOLU] Tâche 16 : queue locale hors-ligne + retry MQTT côté app Chauffeur

**Contexte :** la publication MQTT directe (Tâche 15) fonctionne en ligne, mais toute coupure réseau pendant une tournée active ferait perdre les positions capturées. Étape 0 de la Tâche 16 demandait de confirmer le mécanisme de stockage local.
**Décision Étape 0 :** confirmé par l'utilisateur (`AskUserQuestion`, avant de coder) — `sqflite` (suggéré explicitement par `TASK_PROMPTS.md`), plutôt qu'un fichier JSON Lines persisté à la main.
**Implémentation (dépôt `autombalit-mobile`, branche `feat/chauffeur-queue-hors-ligne`) :**
- `lib/driver/models/position_capturee.dart` : structure de données pure (`id` local nullable, `camionId`, `latitude`, `longitude`, `horodatage`) — pas de logique métier (CONVENTIONS.md §Architecture).
- `lib/driver/repositories/position_queue_repository.dart` (`PositionQueueRepository`) : base SQLite dédiée (`positions_en_attente.db`, table du même nom), CRUD minimal (`ajouter`, `enAttente` triée par `horodatage ASC`, `compterEnAttente`, `supprimer`) — persistance réelle sur disque, survit à un redémarrage de l'app (critère d'acceptation).
- `lib/driver/services/mqtt_position_client.dart` : remplace la publication "fire-and-forget" de la Tâche 15 par `publierEtAttendreAccuse` (attend le PUBACK QoS via le flux `MqttClient.published`, retourne `true` seulement une fois l'ack reçu, `false` sans exception sur timeout/échec/déconnexion) + `estConnecte`/`reconnecterSiNecessaire` (reconnexion best-effort avec les identifiants de la connexion initiale, mémorisés en interne).
- `lib/driver/repositories/tracking_repository.dart` : à chaque cycle de fréquence adaptative (Tâche 15, formule **non modifiée** — Tâche 16 point 5), la position est d'abord persistée (`positionQueueRepository.ajouter`, indépendamment de la connectivité) puis `_traiterFileAttente()` est déclenché — reconnecte si nécessaire, publie la file en attente **dans l'ordre de capture**, en s'arrêtant à la première position non confirmée (retentée en entier au passage suivant, jamais de dépassement d'une position en échec). Un second minuteur périodique (5s, indépendant de la fréquence adaptative de capture) rappelle `_traiterFileAttente()` en continu — c'est lui qui garantit le retry automatique à la reconnexion, y compris si l'app est relancée en cours de tournée avec des positions déjà en file. Un verrou (`_traitementFileEnCours`) évite deux passages concurrents.
- `TrackingProvider.positionsEnAttente` (rafraîchi toutes les 3s pendant le suivi actif) + bandeau ambre dédié sur `TourneeDuJourScreen` (SPEC.md §11, sous le bandeau GPS vert existant) — affiché seulement si `positionsEnAttente > 0`.
**Tests :** `test/driver/position_queue_repository_test.dart` (5 tests) sur une vraie base SQLite via `sqflite_common_ffi` (pas de mock — CONVENTIONS.md §Tests exige des tests réels sur cette logique), couvrant ajout/tri/suppression/persistance entre instances. `test/driver/tracking_repository_test.dart` étendu (persistance + purge sur ack, position non confirmée retentée à la reconnexion) et `test/driver/tracking_provider_test.dart` étendu (rafraîchissement de l'indicateur, remise à zéro à l'arrêt) — mocks `mocktail` avec une file en mémoire simulant la persistance réelle.
**Vérification de bout en bout sur l'appareil physique** (chauffeur de test `+221771234567`, camion `TEST-001`) : tournée démarrée, broker Mosquitto arrêté (`docker compose stop mosquitto`) pendant ~75s → bandeau "4 position(s) en attente d'envoi…" affiché, confirmé qu'aucun nouveau `PositionCamion` n'était créé pendant la coupure. Broker redémarré (`docker compose start mosquitto`) → bandeau disparu quelques secondes plus tard (positions purgées de la file locale, ack MQTT reçu), conforme au critère d'acceptation côté app.
**Bug d'infrastructure découvert (hors périmètre de cette tâche, dépôt `autombalit-backend`) :** le service `mqtt_listener` (Tâche 9) ne se reconnecte jamais automatiquement à Mosquitto après un redémarrage du broker (`Déconnecté du broker MQTT : Unspecified error` en boucle dans ses logs, sans jamais retenter). Conséquence observée pendant ce test : les 4 positions purgées de la file locale à la reconnexion (ack broker reçu côté app) ne sont jamais arrivées dans `PositionCamion`, car aucun abonné backend n'était connecté pour les recevoir au moment de leur republication (MQTT QoS ne rejoue pas les messages à un abonné qui n'était pas connecté au moment de la publication, en l'absence de session persistante). Un `docker compose restart mqtt_listener` manuel a confirmé le diagnostic : les captures suivantes (5 positions) sont arrivées normalement, dans l'ordre, à l'intervalle attendu. **Non corrigé ici** — ce comportement concerne le listener backend (Tâche 9), pas l'app mobile ; signalé pour une tâche corrective dédiée côté `autombalit-backend` (voir `BUGS_AND_ROADMAP.md`).
**Statut :** ✅ Résolu (Tâche 16, périmètre mobile) — bug de reconnexion du `mqtt_listener` backend signalé séparément, non corrigé.

## [RÉSOLU/PARTIEL] Reconnexion automatique du `mqtt_listener` après coupure du broker Mosquitto

**Contexte :** bug signalé lors de la Tâche 16 (`autombalit-mobile`) — voir entrée ci-dessus — 4 positions accusées côté app (PUBACK reçu) après une coupure Mosquitto de ~75s, jamais arrivées dans `PositionCamion`, nécessitant un `docker compose restart mqtt_listener` manuel pour que les captures suivantes reprennent. Branche `fix/mqtt-listener-reconnexion` (`autombalit-backend`).
**Étape 0 :** inspection du code source installé de `paho-mqtt==2.1.0` (`Client.loop_forever`) : la reconnexion automatique après coupure non provoquée est bien native (`reconnect_on_failure=True` par défaut, backoff exponentiel via `reconnect_delay_set`, `min_delay`/`max_delay` par défaut 1s/120s) — pas de boucle de reconnexion maison à écrire. `_handle_connect` (déjà existant, Tâche 9) rappelait déjà `client.subscribe(...)` à chaque `on_connect`, donc le ré-abonnement après reconnexion fonctionnait déjà aussi.

**Bug confirmé et corrigé — invisibilité des logs :** le projet n'a aucune config `LOGGING` Django, donc la racine du module `logging` n'a aucun handler et retombe sur le "handler de dernier recours" de Python, qui ignore tout en dessous de `WARNING`. Seul le `logger.warning(...)` de déconnexion apparaissait dans `docker compose logs` ; tout `logger.info(...)` (dont le log de reconnexion réussie) était silencieusement perdu. Corrigé par l'ajout d'une config `LOGGING` minimale dans `settings.py` (handler console, niveau `INFO`).

**Cause exacte du symptôme original — non établie avec certitude, malgré investigation :**
- Un premier test en conditions réelles (`docker compose stop/start mosquitto`, coupure ~35s, **avec le fix de backoff déjà appliqué**) a montré une reconnexion propre et rapide, ce qui a d'abord conduit à une conclusion erronée : que la reconnexion native fonctionnait déjà correctement même *avant* le fix, et que seule l'invisibilité des logs expliquait le symptôme de la Tâche 16.
- Remise en question à juste titre : un problème de logs invisibles n'explique pas à lui seul pourquoi des positions n'ont *jamais* atteint la base. Un test A/B a donc été mené en revenant temporairement au code **d'avant le fix** (backoff par défaut de paho, non borné, jusqu'à 120s) sur une coupure de ~97s (comparable aux ~75s du test terrain) : le listener s'est reconnecté 5,7s seulement après le retour du broker (déconnexion détectée à T+0s, broker de nouveau sain à T+97,3s, reconnexion effective à T+103s). **Ce test contredit directement l'hypothèse que le backoff par défaut, à lui seul, explique un blocage prolongé** — dans cette reproduction, il ne l'a pas fait.
- Une troisième piste a été identifiée mais **non vérifiée** : le `keepalive` MQTT n'est jamais configuré explicitement (défaut paho = 60s). Si la coupure originale n'a pas produit de FIN/RST TCP immédiat côté client (contrairement aux deux reproductions ci-dessus, où la déconnexion a été détectée en ~1s après `docker compose stop`), le client aurait pu rester dans un état "connecté" illusoire jusqu'à expiration du keepalive (~1,5× 60s ≈ 90s) avant même de commencer à retenter sa connexion — ce qui, cumulé à la coupure elle-même, pourrait couvrir toute la fenêtre du test terrain. Non testé (aurait nécessité de simuler une coupure réseau sans FIN propre, ex. règle `iptables`/`docker network disconnect`, non tenté par manque de temps/priorité).
- Les journaux bruts de la session de test terrain de la Tâche 16 (dépôt `autombalit-mobile`) ne sont plus disponibles pour ré-examen — seule la synthèse déjà consignée dans l'entrée Tâche 16 ci-dessus subsiste.
- **Conclusion : la cause exacte de la perte de positions observée lors du test terrain de la Tâche 16 ne peut plus être établie avec certitude a posteriori.** Ni "le backoff par défaut a causé un blocage" (infirmé par la reproduction), ni "les positions sont en réalité arrivées mais n'ont pas été revérifiées" (peu plausible — l'entrée Tâche 16 documente une vérification active à l'époque) ne sont des explications démontrées. L'hypothèse keepalive reste ouverte mais non prouvée.

**Fix appliqué (`tracking/clients/mqtt_client.py`, `autombalit_backend/settings.py`) — justifié comme durcissement, pas comme correction démontrée du mécanisme exact :**
- `PositionMQTTClient.__init__` appelle désormais explicitement `reconnect_delay_set(min_delay=1, max_delay=30)` — backoff borné à 30s (au lieu des 120s par défaut non documentés). Réduit le pire cas possible, même si la reproduction n'a pas montré que le défaut posait problème dans ce scénario précis.
- `_handle_connect` distingue connexion initiale (`logger.info`) et reconnexion après coupure (`logger.warning`, plus visible), toujours suivi du ré-abonnement à `camions/+/position`.
- `_handle_disconnect` log désormais explicitement qu'une reconnexion automatique est en cours avec son backoff plafonné.
- `settings.py` : config `LOGGING` minimale (handler console, niveau `INFO` sur la racine) — corrige la cause confirmée (invisibilité des logs), au passage pour tous les `logger.info(...)` du projet.
**Vérification :** 10 tests (`tracking/tests/test_mqtt_client.py`, dont 5 nouveaux) sur le backoff configuré et le contenu/niveau des logs connect/disconnect/reconnect ; suite complète (117 tests) au vert. Trois tests de bout en bout sur la vraie stack Docker Compose : coupure ~35s post-fix (reconnexion + persistance confirmées), coupure ~97s pré-fix en A/B (reconnexion en 5,7s après retour du broker, backoff par défaut non mis en cause), coupure ~75s+ post-fix avec log explicite de reconnexion et position republiée confirmée persistée en base (camion `TEST-001`, pk=4), sans aucune intervention manuelle sur `mqtt_listener` dans tous les cas testés.
**Leçon :** ne pas confondre "le comportement actuel semble correct" et "j'ai identifié la cause du bug original" — un fix peut être un durcissement légitime (réduire un pire cas théorique, corriger un vrai bug de visibilité trouvé en cours de route) sans que la cause exacte du symptôme initialement rapporté soit démontrée. Le dire explicitement plutôt que documenter une explication non vérifiée comme si elle était établie.
**Statut :** ✅ Résolu pour le bug d'invisibilité des logs (confirmé) et durci pour la reconnexion (backoff borné, re-abonnement loggé) — ⚠️ cause exacte du symptôme original de la Tâche 16 non établie avec certitude ; à rouvrir si une perte de positions similaire est de nouveau observée en pilote malgré ce fix (auquel cas creuser l'hypothèse keepalive en premier).

## [CHOIX] Limite structurelle : un ack MQTT (PUBACK) ne garantit qu'une livraison au broker, jamais une persistance en base

**Contexte :** mise en lumière par le test de la Tâche 16 (queue hors-ligne chauffeur, voir entrée dédiée ci-dessus) et le bug de reconnexion `mqtt_listener` corrigé juste au-dessus — les deux ont montré concrètement qu'une position dont l'app chauffeur reçoit le PUBACK (Tâche 16, `publierEtAttendreAccuse`) peut malgré tout ne jamais atteindre `PositionCamion` si `mqtt_listener` n'était pas connecté/abonné au moment de la republication (MQTT ne rejoue pas les messages à un abonné absent, en l'absence de session persistante côté client backend).
**Cause :** c'est le contrat réel de MQTT QoS 1/2 tel qu'utilisé ici — le PUBACK confirme uniquement que le **broker** a reçu et retenu le message pour distribution aux abonnés *alors connectés*, jamais qu'un abonné donné (ici `mqtt_listener`) l'a effectivement reçu et traité. L'app chauffeur n'a structurellement aucun moyen de distinguer "livré au broker" de "persisté en base" avec un simple ack QoS.
**Décision :** accepter cet écart pour le MVP pilote plutôt que de le combler maintenant — le fix de reconnexion ci-dessus réduit fortement sa fenêtre d'exposition (le listener ne reste plus durablement déconnecté en silence), ce qui suffit pour la Phase 5 (test pilote). Ne pas complexifier la chaîne de bout en bout avec un accusé applicatif tant que le volume réel de coupures observées en pilote ne justifie pas l'effort.
**Amélioration future si nécessaire (non implémentée ici) :** accusé applicatif de bout en bout — ex. le backend republie un ack dédié (WebSocket/MQTT retour, ou endpoint HTTP de confirmation) une fois la position réellement persistée en base, que l'app chauffeur attend avant de purger définitivement sa file locale (au lieu de purger dès le PUBACK MQTT). À évaluer via `BUGS_AND_ROADMAP.md` si des pertes de positions sont à nouveau constatées en pilote malgré le fix de reconnexion.
**Statut :** 🔵 Choix assumé — écart documenté, non comblé pour le MVP pilote.

## [CHOIX] Gouvernance de démarrage : scénario C (portage solo)

**Contexte :** trois scénarios de portage possibles (mairie, société privée, plateforme indépendante).
**Décision :** démarrer en scénario C (MVP porté en solo, quartier pilote + société ou camion volontaire) pour prouver le concept, avant d'envisager un partenariat institutionnel ou privé plus large.
**Statut :** 🔵 Choix assumé — à réévaluer après le test pilote (Phase 5)

## [RÉSOLU] Complément backend Tâche 17 : endpoint `/api/points-enregistres/`

**Contexte :** l'Étape 0 de la Tâche 17 (`autombalit-mobile`) devait s'appuyer sur `POST /api/points-enregistres/` (SPEC.md §7, cité dans le Constat de la tâche comme déjà fourni). Vérification du code réel avant de coder côté Flutter : le modèle `PointEnregistre` existe (`citizens/models.py`) mais aucun serializer/viewset/route ne l'exposait — `citizens/views.py` était resté un placeholder Django par défaut depuis le scaffolding initial. Signalé à l'utilisateur avant de coder (`AskUserQuestion`, pattern déjà anticipé par la Tâche 15 pour un cas similaire).
**Décision :** confirmé par l'utilisateur — ajouter le complément backend maintenant plutôt que de différer, pour que le critère d'acceptation de la Tâche 17 reste vérifiable de bout en bout. Sur une branche dédiée (`feat/citoyen-endpoint-points-enregistres`, `autombalit-backend`), en suivant les conventions déjà établies (Tâche 4 : DRF centralisé dans l'app `api`, pas dans l'app métier propriétaire du modèle) :
- `citizens/services/points_enregistres.py` : `deduire_zone(point)` (recherche la `Zone` dont le polygone contient le point, `None` sinon) et `creer_point_enregistre(...)` — logique métier non triviale (CONVENTIONS.md §Validation : un point hors de toute zone connue reste exploitable mais sans zone associée) isolée du serializer/de la vue.
- `api/permissions.py` : `EstCitoyen`, symétrique à `EstChauffeurValide` — vérifie `request.user.utilisateur`, aucune validation manuelle requise contrairement au chauffeur (SPEC.md §4.3).
- `api/serializers.py` : `PointEnregistreSerializer` (`point` en GeoJSON via `GeometryField`, cohérent avec `ZoneSerializer` ; `zone` en lecture seule, `{id, nom}` ou `None`).
- `api/views.py`/`api/urls.py` : `PointEnregistreViewSet` (`ModelViewSet` restreint à `get/post/delete`, SPEC.md §7 ne prévoyant pas de PUT/PATCH), scope systématique sur `request.user.utilisateur.points_enregistres`.
**Vérification :** 11 tests nouveaux (`citizens/tests/test_points_enregistres.py` : déduction de zone ; `api/tests/test_points_enregistres.py` : liste/création/suppression scopées au citoyen connecté, zone déduite ou `None`, chauffeur/anonyme rejetés) ; suite complète (128 tests) au vert ; `manage.py check` propre. Pas encore commité (accord explicite requis avant tout commit, `autombalit-backend`).
**Statut :** ✅ Résolu (code + tests) — commit en attente d'accord.

## [RÉSOLU] Tâche 17 : app Citoyen — auth OTP partagée + enregistrement du domicile

**Contexte :** premier écran de l'app citoyen au-delà du placeholder (Tâche 3). Étape 0 : partager l'écran OTP avec le flavor chauffeur (`lib/common`) ou le dupliquer par flavor — confirmé par l'utilisateur (`AskUserQuestion`) : partage, le flux Firebase étant identique et la palette déjà gérée au niveau `MaterialApp` de chaque flavor.
**Décision (architecture)** :
- `AuthProvider`, `AuthRepository` et `OtpScreen` déplacés de `lib/driver/` vers `lib/common/` (providers/repositories/screens) — `AuthRepository` reçoit désormais un `role` (`'chauffeur'`/`'citoyen'`) au lieu du littéral `'chauffeur'` codé en dur, transmis tel quel à `POST /api/auth/token/` ; `statut_validation` devient nullable de bout en bout (`String?`) puisqu'absent de la réponse backend pour un citoyen. `OtpScreen` reçoit un `titre` (`'Connexion chauffeur'`/`'Connexion citoyen'`) au lieu d'un texte figé. `DriverApp` mis à jour en conséquence (imports, `role: 'chauffeur'`, `titre: 'Connexion chauffeur'`) — comportement chauffeur inchangé, seulement relocalisé.
- Nouveau module `lib/citizen/` : `models/point_enregistre.dart`, `services/localisation_service.dart` (permission + relevé ponctuel `geolocator`, distinct de `PositionStreamService` chauffeur dont l'usage — suivi continu pendant tournée — diffère), `repositories/points_enregistres_repository.dart`, `providers/enregistrement_domicile_provider.dart`, `screens/enregistrement_domicile_screen.dart`, `citizen_app.dart` (miroir de `DriverApp`, sans gate de statut de validation). `main_citoyen.dart` réécrit sur le modèle de `main_chauffeur.dart` (init Firebase + `runApp(CitizenApp())`).
**Décision (produit, non tranchée explicitement avec l'utilisateur, assumée)** : « pointage direct sur la carte » retenu plutôt que « recherche d'adresse » (les deux étaient permis par l'énoncé, connecté par « ou ») — repère fixe au centre de l'écran représentant `mapController.camera.center`, déplacé en déplaçant la carte ; bouton « Me localiser » (icône, `geolocator`) pour recentrer sur la position actuelle. Évite une dépendance à un service de géocodage tiers (Nominatim/OSM ou autre) non budgétée et non choisie. `flutter_map ^8.3.2` + `latlong2` ajoutés à `pubspec.yaml` — tuiles OSM publiques (`tile.openstreetmap.org`), cohérent avec SPEC.md §2/§10 (pas de SDK cartographique payant) ; `flutter_map` avertit en test/exécution que ces tuiles publiques ne sont pas prévues pour un usage production à fort volume — point de vigilance à revisiter en Phase 5 (test pilote réel), au même titre que le profil OSRM `car.lua` (Tâche 6).
**Vérification :**
- `flutter analyze` propre ; `flutter test` : 32 tests au vert (dont 7 nouveaux `test/citizen/` — provider : succès/zone nulle/erreur API ; écran : rendu carte+formulaire, validation nom vide, confirmation après enregistrement — et déplacement de `test/driver/auth_provider_test.dart`/`otp_screen_test.dart` vers `test/common/` avec mise à jour des imports).
- `flutter build apk --flavor citoyen --debug` réussi (mêmes avertissements Gradle/AGP/Kotlin déjà documentés Tâche 3, non bloquants).
- **Test partiel sur l'appareil physique** (Samsung, `adb devices` déjà autorisé) : APK installée et lancée avec succès, écran « Connexion citoyen » affiché avec la palette verte attendue, aucun crash au démarrage. L'automatisation de la saisie clavier via `adb shell input text` n'a pas fonctionné sur ce clavier Samsung HoneyBoard (le champ obtient le focus mais le texte injecté n'apparaît jamais, sans erreur) — non creusé davantage (hors périmètre de la tâche mobile). **Le test interactif complet (OTP réel avec le numéro de test Firebase `+221771234567`/code fixe `123456` — voir `FIXTURES_TEST.md` —, pointage sur la carte, soumission, vérification via `GET /api/points-enregistres/`) reste à faire manuellement par l'utilisateur**, comme pour la première vérification OTP de la Tâche 14.
**Non fait dans cette tâche** (hors périmètre annoncé) : recherche d'adresse (voir décision produit ci-dessus), notifications FCM (Tâche 18), écran statut du jour (Tâche 19), carte live (Tâche 22).
**Statut :** ✅ Résolu (code + tests + vérification partielle sur appareil) — test interactif complet en attente de confirmation par l'utilisateur ; commit en attente d'accord (`autombalit-mobile`).
