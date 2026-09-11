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

## [CHOIX] Gouvernance de démarrage : scénario C (portage solo)

**Contexte :** trois scénarios de portage possibles (mairie, société privée, plateforme indépendante).
**Décision :** démarrer en scénario C (MVP porté en solo, quartier pilote + société ou camion volontaire) pour prouver le concept, avant d'envisager un partenariat institutionnel ou privé plus large.
**Statut :** 🔵 Choix assumé — à réévaluer après le test pilote (Phase 5)
