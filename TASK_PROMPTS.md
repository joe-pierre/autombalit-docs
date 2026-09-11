# Prompts de tâches — AutoMbalit

Chaque tâche ci-dessous est rédigée pour être copiée telle quelle dans Claude Code. Aucune tâche ne doit être lancée sans relecture préalable. Claude Code ne doit committer que sur instruction explicite ("commit").

**Mécanique Git multi-dépôts** : le projet est réparti sur trois dépôts séparés (`autombalit-docs`, `autombalit-backend`, `autombalit-mobile`) dans un dossier parent commun. Avant de coder, se positionner (`cd`) dans le dépôt indiqué par le champ "Branche" de la tâche et y créer/basculer la branche. Le commit du code (dans `autombalit-backend` ou `autombalit-mobile`) et la mise à jour de `TODO.md`/`DECISIONS.md` (dans `autombalit-docs`, prévue en fin de tâche) sont **deux commits séparés, dans deux dépôts différents** — ne pas les confondre en un seul commit. "commit" seul = commit local uniquement ; préciser "commit et push" pour pousser vers GitHub dans la foulée.

---

## TÂCHE 1 — Setup du projet Django + PostGIS + GeoDjango

Contexte : projet AutoMbalit, nouvelle branche. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/setup-backend-django-postgis`

Constat
Aucun code n'existe encore. Il faut poser les fondations backend : projet Django, connexion PostgreSQL/PostGIS, activation de GeoDjango, structure des apps définie dans `SPEC.md` §5.

Étape 0 — Avant de coder
- Confirmer la version de Django et Python à utiliser (Django 5.x, Python 3.11+ recommandé).
- Vérifier que PostGIS est disponible dans l'environnement cible (local + VPS de déploiement).

TÂCHE
1. Initialiser le projet Django (`autombalit_backend`) avec les apps suivantes, vides pour l'instant : `core`, `tracking`, `citizens`, `notifications`, `realtime`, `geo_import`, `api` (voir `SPEC.md` §5).
2. Configurer `settings.py` pour PostgreSQL/PostGIS (`django.contrib.gis.db.backends.postgis`), activer `django.contrib.gis` dans `INSTALLED_APPS`.
3. Mettre en place la gestion des variables d'environnement (`.env` non commité, `.env.example` fourni) pour les secrets (DB, futur Firebase).
4. Ajouter Django REST Framework et `djangorestframework-simplejwt` aux dépendances, sans encore créer d'endpoints.
5. Fournir un `requirements.txt` (ou `pyproject.toml` si tu préfères Poetry — préciser le choix) et un `docker-compose.yml` minimal pour PostgreSQL/PostGIS en local.
6. Ne pas encore créer de modèles métier — cette tâche se limite au squelette du projet.

Contraintes
- Respecter la stack validée dans `SPEC.md` §2, aucune déviation (pas de MySQL, pas de SQLite en cible).
- Respecter les conventions de nommage de `CONVENTIONS.md`.
- Respecter dès le squelette l'architecture en couches définie dans `CONVENTIONS.md` §Architecture et paradigmes (dossiers `services/`, `selectors/`, `clients/` prévus par app même vides à ce stade) — ne pas tout mettre dans `views.py`/`models.py` par défaut.
- Pas de secret en dur dans le code.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme la version Python/Django et le choix `requirements.txt` vs `pyproject.toml`.
- En fin de tâche : liste les commandes à exécuter pour lancer le projet en local (migrations, runserver, docker-compose up).

Critère d'acceptation
Le projet Django démarre sans erreur, la connexion à une base PostgreSQL/PostGIS locale (via docker-compose) fonctionne, `python manage.py check` ne remonte aucune erreur liée à GeoDjango.

---

## TÂCHE 2 — Modèles de données et migrations

Contexte : projet AutoMbalit, suite de la Tâche 1. Se référer au modèle de données validé dans `SPEC.md` §3 et aux modèles Django déjà produits en conception.

Branche : `feat/models-core-tracking-citizens`

Constat
Le squelette du projet existe (Tâche 1) mais aucun modèle métier n'est encore implémenté. Il faut créer les modèles `Societe`, `Chauffeur`, `Camion`, `Zone`, `Tournee`, `TourneeZone`, `CalendrierCollecte`, `AssignationJournaliere`, `PositionCamion`, `Utilisateur`, `PointEnregistre`, `Signalement`, répartis dans les apps `core`, `tracking`, `citizens` selon `SPEC.md` §5.

Étape 0 — Avant de coder
- Relire le schéma entité-relation et les modèles Django déjà validés (voir historique de conception / `SPEC.md` §3).
- Confirmer la répartition des modèles par app : `core` (Societe, Chauffeur, Camion, Zone, Tournee, TourneeZone, CalendrierCollecte), `tracking` (AssignationJournaliere, PositionCamion), `citizens` (Utilisateur, PointEnregistre, Signalement).

TÂCHE
1. Implémenter les modèles Django (GeoDjango) exactement selon la structure validée dans `SPEC.md` §3 — champs, types de géométrie (`PointField`, `PolygonField`, `LineStringField`, SRID 4326), clés étrangères, contraintes `unique_together`.
2. Ajouter les index nécessaires, en particulier `PositionCamion (camion, -horodatage)`.
3. Générer les migrations et vérifier qu'elles s'appliquent proprement sur une base PostGIS vide.
4. Enregistrer les modèles dans l'admin Django (admin natif, pas le futur web admin métier) pour permettre une inspection manuelle rapide pendant le développement.
5. Ne pas créer de serializers ni d'endpoints API dans cette tâche — modèles et migrations uniquement.

Contraintes
- Respecter strictement les noms de champs français déjà définis (`horodatage`, pas `timestamp`, etc.) — voir `CONVENTIONS.md`.
- Toute géométrie doit être en SRID 4326.
- Contrainte unique `(camion, date)` sur `AssignationJournaliere` au niveau base de données, pas seulement applicatif.
- Les modèles restent des structures de données (champs, contraintes, propriétés simples) — aucune logique métier (calcul ETA, seuils, validation GeoJSON) dans les méthodes de modèle : voir `CONVENTIONS.md` §Architecture et paradigmes (règle anti god-model, en particulier sur `PositionCamion`).

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme que la répartition des modèles par app proposée ci-dessus te convient, ou propose une alternative justifiée.
- En fin de tâche : mets à jour `DECISIONS.md` si un écart au modèle de conception initial a été nécessaire, et coche les cases correspondantes dans `TODO.md` (Phase 1).

Critère d'acceptation
`python manage.py makemigrations` puis `migrate` s'exécutent sans erreur sur une base PostGIS vide ; tous les modèles sont visibles et manipulables depuis l'admin Django ; les contraintes d'unicité définies sont vérifiables (test manuel ou test automatisé simple).

## TÂCHE 3 — Flavors Flutter (citoyen / chauffeur) et structure de dossiers

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Le projet a été scaffoldé via `flutter create` (voir `GUIDE_DU_DEVELOPPEUR.md` §2.3) — une seule codebase Flutter, mais aucune séparation citoyen/chauffeur n'est encore configurée. Se référer à `SPEC.md` §5 (architecture mobile) et §11 (écrans) avant toute modification.

Branche : `feat/flutter-flavors-citoyen-chauffeur`

Constat
Le projet Flutter existe à l'état de squelette généré par défaut (un seul `lib/main.dart`, aucun flavor Android). Il faut mettre en place les deux flavors (`citoyen`, `chauffeur`) et la structure de dossiers validée dans `SPEC.md` §5, sans encore implémenter de logique métier.

Étape 0 — Avant de coder
- Proposer et confirmer avec moi le schéma de nommage des `applicationId` Android pour les deux flavors (ex. `com.autombalit.citoyen` / `com.autombalit.chauffeur`), car il ne pourra plus être changé facilement une fois l'app publiée.
- Confirmer que la structure de dossiers proposée dans `SPEC.md` §5 (`lib/common/`, `lib/driver/`, `lib/citizen/`, `lib/shared_widgets/`) est toujours celle à suivre.

TÂCHE
1. Configurer deux flavors Android dans `android/app/build.gradle` : `citoyen` et `chauffeur`, avec `applicationId` distincts et un nom d'app (label) distinct par flavor.
2. Créer deux points d'entrée : `lib/main_citoyen.dart` et `lib/main_chauffeur.dart`, chacun lançant un `MaterialApp` minimal avec un écran placeholder clairement identifié (ex. `Text("App Citoyen")` / `Text("App Chauffeur")`) — pas de logique métier, juste la preuve que le bon flavor se lance.
3. Créer la structure de dossiers définie dans `SPEC.md` §5 : `lib/common/`, `lib/driver/`, `lib/citizen/`, `lib/shared_widgets/`, avec un fichier `.gitkeep` ou un placeholder minimal dans chacun s'il est vide.
4. Vérifier que les deux flavors se lancent correctement sur le téléphone physique (voir `GUIDE_DU_DEVELOPPEUR.md` §2.7) :
   - `flutter run --flavor citoyen -t lib/main_citoyen.dart -d <device_id>`
   - `flutter run --flavor chauffeur -t lib/main_chauffeur.dart -d <device_id>`
5. Ne pas implémenter : authentification Firebase, géolocalisation, MQTT, appels API, ou tout autre écran au-delà du placeholder. Cette tâche se limite au scaffolding des flavors et de la structure.

Contraintes
- Respecter `CONVENTIONS.md` §Architecture et paradigmes : aucune logique métier dans les widgets, dossiers `services/`, `repositories/`, gestion d'état séparés préparés dès la structure (même vides à ce stade).
- Android uniquement pour l'instant (voir `DECISIONS.md` — iOS repoussé après validation MVP) : ne pas configurer de flavors iOS dans cette tâche.
- Aucun secret ni configuration Firebase réelle codée en dur dans cette tâche — la config Firebase (`flutterfire configure`) reste une étape manuelle séparée (voir `GUIDE_DU_DEVELOPPEUR.md` §2.4), non refaite par Claude Code.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme le schéma de nommage des `applicationId` proposé à l'Étape 0.
- En fin de tâche : coche la case correspondante dans `TODO.md` (Phase 3), et ajoute une entrée dans `DECISIONS.md` si le schéma de nommage des `applicationId` ou la structure de dossiers a dû s'écarter de ce qui était prévu dans `SPEC.md`.

Critère d'acceptation
Les deux flavors se compilent et se lancent sans erreur sur le téléphone physique via les commandes `flutter run --flavor ...` ci-dessus, chacun affichant son écran placeholder distinct confirmant que le bon flavor a été chargé ; la structure de dossiers `lib/common/`, `lib/driver/`, `lib/citizen/`, `lib/shared_widgets/` existe et est commitée.

---

## TÂCHE 4 — Endpoints DRF CRUD zones/tournées + auth JWT

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/api-crud-zones-tournees-auth-jwt`

Constat
Les modèles (Tâche 2) et le squelette Django (Tâche 1) existent, `djangorestframework-simplejwt` est dans les dépendances mais non configuré. Aucun serializer, viewset ni route API n'existe. L'app `api` est vide.

Étape 0 — Avant de coder
- Confirmer la répartition des serializers/viewsets : tout dans l'app `api`, ou répartis par app métier (`tracking`, `geo_import`) avec `api` en simple couche de routing.
- Confirmer qu'aucune granularité de permission par rôle (société / admin / chauffeur) n'est encore requise à ce stade — seulement `IsAuthenticated`.

TÂCHE
1. Configurer JWT : `REST_FRAMEWORK.DEFAULT_AUTHENTICATION_CLASSES` avec `JWTAuthentication`, routes `/api/token/` et `/api/token/refresh/` via simplejwt.
2. Créer les serializers pour `Zone` et `Tournee` (et `TourneeZone` si nécessaire), sans logique métier dans le serializer.
3. Créer des viewsets DRF CRUD pour `Zone` et `Tournee`, permission `IsAuthenticated` uniquement pour l'instant.
4. Déclarer les routes via un router DRF dans `api/urls.py`, monté dans les urls du projet.
5. Ne pas implémenter : l'upload GeoJSON (Tâche 5), la validation métier avancée, les endpoints tracking/MQTT, l'auth OTP Firebase (Tâche 12).

Contraintes
- Respecter les noms de champs français déjà définis dans les modèles.
- Architecture en couches (`CONVENTIONS.md`) : serializers minces, logique dans `services/` si besoin.
- Un test automatique minimum par endpoint CRUD (création, lecture, liste) via le client de test DRF.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme la répartition des serializers/viewsets et la stratégie de permission par défaut.
- En fin de tâche : coche la case correspondante dans `TODO.md` (Phase 1), ajoute une entrée dans `DECISIONS.md` en cas d'écart.

Critère d'acceptation
Les endpoints CRUD zones/tournées sont accessibles avec un JWT valide obtenu via `/api/token/`, renvoient 401 sans token ; les tests automatiques passent.

---

## TÂCHE 5 — Upload GeoJSON + validation à l'upload

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/api-upload-geojson-validation`

Constat
Les zones sont censées provenir de GeoJSON pré-converti par l'équipe SIG (`SPEC.md`). Aucun endpoint d'upload ni logique de validation n'existe encore.

Étape 0 — Avant de coder
- Confirmer le schéma exact des `properties` GeoJSON attendues par zone (nom, société associée, etc. — voir `SPEC.md`) et le(s) type(s) de géométrie accepté(s) (Polygon/MultiPolygon).
- Proposer et confirmer la stratégie d'import : all-or-nothing (transaction atomique) par défaut, sauf avis contraire.

TÂCHE
1. Créer un endpoint d'upload GeoJSON (app `geo_import`) qui crée ou met à jour les `Zone` correspondantes.
2. Implémenter la validation dans un service dédié (`geo_import/services/`) : SRID 4326 explicite, type de géométrie conforme, `properties` requises présentes et bien typées.
3. Retourner des erreurs de validation localisées (quelle feature du GeoJSON pose problème), pas une erreur générique.
4. Implémenter la stratégie d'import confirmée à l'Étape 0 (transaction atomique par défaut).
5. Ne pas implémenter : l'interface web d'upload (Phase 4) — cette tâche fournit uniquement l'endpoint API.

Contraintes
- Toute géométrie stockée en SRID 4326.
- Rejeter plutôt que corriger silencieusement une géométrie ambiguë.
- Tests : GeoJSON valide, SRID absent/différent, type de géométrie invalide, `properties` manquantes.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le schéma `properties` attendu et la stratégie d'import.
- En fin de tâche : coche `TODO.md` (Phase 1), documente dans `DECISIONS.md` la stratégie retenue et le schéma `properties`.

Critère d'acceptation
L'endpoint accepte un GeoJSON valide et crée les `Zone` correspondantes ; il rejette avec un message clair tout GeoJSON non conforme (SRID, géométrie, properties) ; les tests automatiques passent.

---

## TÂCHE 6 — Préparation des données OSRM + intégration docker-compose

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `GUIDE_DU_DEVELOPPEUR.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/osrm-data-prep`

Constat
Le service `osrm` est déjà déclaré en placeholder dans `docker-compose.yml` (Tâche 1) mais aucune donnée OSM n'a été préparée : pas d'extrait Geofabrik téléchargé, pas de pipeline `osrm-extract`/`partition`/`customize` exécuté.

Étape 0 — Avant de coder
- Confirmer le périmètre de l'extrait à utiliser (Sénégal entier vs région Dakar), le quartier pilote n'étant pas encore choisi (`TODO.md` Phase 0).
- Confirmer l'usage du profil `car.lua` par défaut pour l'instant, en documentant "camion ≈ voiture" comme approximation connue (précision réelle à valider en Phase 5).

TÂCHE
1. Documenter et scripter le téléchargement de l'extrait OSM depuis Geofabrik (pas au moment du build de l'image, pour rester léger).
2. Exécuter le pipeline OSRM (extract/partition/customize) et produire les fichiers `.osrm*` nécessaires.
3. Adapter le service `osrm` dans `docker-compose.yml` pour monter ces données et lancer `osrm-routed` dessus.
4. Vérifier via une requête `/route/v1/driving/...` que le service répond correctement une fois le stack démarré.
5. Documenter dans `GUIDE_DU_DEVELOPPEUR.md` la procédure de régénération des données OSRM.
6. Ne pas implémenter : l'appel applicatif Django → OSRM pour le calcul d'ETA (Tâche 10).

Contraintes
- Pas de service payant (Geofabrik + OSRM self-hosted uniquement).
- Les fichiers `.osrm*` ne sont pas commités (taille) — `.gitignore` approprié, procédure de génération documentée à la place.
- Profil de routing = voiture par défaut, limite documentée comme point ouvert.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le périmètre de l'extrait et le profil OSRM.
- En fin de tâche : coche `TODO.md` (Phase 1), ajoute une entrée `DECISIONS.md` sur le choix d'extrait/profil.

Critère d'acceptation
`docker compose up` démarre un service `osrm` fonctionnel répondant à une requête de routing test sur la zone choisie ; la procédure de régénération est documentée.

---

## TÂCHE 7 — Validation de la stack complète en local

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `GUIDE_DU_DEVELOPPEUR.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/docker-compose-stack-validation`

Constat
Les services (db, redis, mosquitto, osrm, backend) ont chacun été vérifiés séparément au fil des tâches précédentes, mais l'ensemble n'a pas été validé comme un tout cohérent via un seul `docker compose up`.

Étape 0 — Avant de coder
- Lister les services attendus dans `docker-compose.yml` et l'état connu de chacun (fonctionnel isolément ou non) d'après les tâches précédentes, avant de chercher les problèmes d'intégration.

TÂCHE
1. Lancer `docker compose up` sur l'ensemble des services et corriger les problèmes d'orchestration détectés (`depends_on`, healthchecks, variables d'environnement manquantes, ports en conflit).
2. Ajouter des healthchecks Docker Compose pour les services critiques (db, redis, mosquitto, osrm) si absents.
3. Vérifier que le backend Django se connecte correctement à PostGIS et Redis dans ce contexte multi-conteneurs.
4. Documenter dans `GUIDE_DU_DEVELOPPEUR.md` la commande de démarrage complète et l'ordre de vérification en cas de problème.
5. Ne pas implémenter : le service `mqtt_listener` applicatif (Tâche 9) — seulement s'assurer qu'il démarre sans planter s'il existe déjà en placeholder, sinon noter son absence comme non bloquante ici.

Contraintes
- Aucun service ne doit nécessiter une intervention manuelle après `docker compose up` pour être fonctionnel (hors préparation OSRM déjà faite en Tâche 6, hors migrations Django qui peuvent rester une étape documentée séparée).
- Ne pas introduire de nouveaux services hors stack validée.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : partage le résultat d'un premier `docker compose up` (logs, erreurs) avant de corriger.
- En fin de tâche : coche `TODO.md` (Phase 1, dernière case) — ce qui clôture la Phase 1.

Critère d'acceptation
`docker compose up` démarre tous les services sans erreur, le backend est accessible et connecté à PostGIS/Redis, la procédure est documentée dans `GUIDE_DU_DEVELOPPEUR.md`.

---

## TÂCHE 8 — Broker MQTT : authentification par chauffeur + ACL par topic

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/mqtt-broker-auth-acl`

Constat
Le service `mosquitto` existe dans `docker-compose.yml` (Tâche 1) mais sans authentification ni ACL configurées. Aucun schéma de topics n'est encore formellement défini.

Étape 0 — Avant de coder
- Proposer et confirmer un schéma de topics MQTT (ex. `camions/<camion_id>/position`) s'il n'est pas déjà fixé dans `SPEC.md`.
- Confirmer si les credentials MQTT sont liés au chauffeur ou au camion.

TÂCHE
1. Configurer Mosquitto avec authentification par mot de passe (fichier passwd ou plugin), un jeu de credentials par chauffeur/camion.
2. Définir le schéma de topics MQTT confirmé à l'Étape 0.
3. Configurer les ACL Mosquitto pour qu'un chauffeur ne puisse publier que sur son propre topic (pas lire/écrire ceux des autres).
4. Documenter dans `GUIDE_DU_DEVELOPPEUR.md` la procédure de provisioning d'un nouveau chauffeur (création de credentials MQTT), en lien avec le statut `en_attente → validé`.
5. Ne pas implémenter : le handler Django de consommation (Tâche 9), la publication côté app Flutter chauffeur (Phase 3).

Contraintes
- Mosquitto self-hosted uniquement, pas de service MQTT cloud payant.
- QoS 1 ou 2 pour les topics de position.
- Aucun credential en clair dans le dépôt (génération scriptée, exemple dans `.env.example` uniquement).

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le schéma de topics et la stratégie de provisioning des credentials.
- En fin de tâche : coche `TODO.md` (Phase 2), documente le schéma de topics dans `DECISIONS.md` ou `SPEC.md`.

Critère d'acceptation
Un client MQTT authentifié avec les credentials d'un chauffeur peut publier uniquement sur son topic assigné ; une tentative sur un autre topic est refusée ; vérifiable via `mosquitto_pub`/`mosquitto_sub`.

---

## TÂCHE 9 — Handler Django : consommation MQTT → `PositionCamion`

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/mqtt-position-handler`

Constat
Le broker MQTT est fonctionnel et sécurisé (Tâche 8), mais aucun composant applicatif ne consomme les messages pour les persister en base.

Étape 0 — Avant de coder
- Confirmer le mécanisme de consommation : process séparé `mqtt_listener` (déjà prévu dans `docker-compose.yml`) plutôt qu'un consumer Django Channels.

TÂCHE
1. Implémenter le service `mqtt_listener`, abonné aux topics de position de tous les camions actifs.
2. À réception d'un message, créer un `PositionCamion` via un service applicatif dédié (le listener reste mince : parsing/dispatch uniquement).
3. Gérer les rafales de positions (reconnexion après hors-ligne) : persister toutes les positions reçues, sans déclencher de traitement temps réel pour les positions historiques (le traitement temps réel — Tâche 11 — ne portera que sur la plus récente).
4. Gérer les payloads malformés sans faire planter le listener (log + skip).
5. Ne pas implémenter : le calcul ETA (Tâche 10), les seuils/notifications (Tâche 11).

Contraintes
- Architecture en couches : listener mince, logique de persistance dans un service.
- Le tracking n'est accepté que pour une tournée active (start/stop explicite) — rejeter/logguer toute position hors tournée active.
- Tests : simuler une publication MQTT et vérifier la création du `PositionCamion`.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme l'architecture retenue pour le listener.
- En fin de tâche : coche `TODO.md` (Phase 2).

Critère d'acceptation
Un message MQTT publié sur le topic d'un camion en tournée active crée le `PositionCamion` correspondant ; un message hors tournée active est rejeté sans écriture ; les tests automatiques passent.

---

## TÂCHE 10 — Intégration OSRM (calcul distance/temps) + facteur de correction historique

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/osrm-eta-calcul`

Constat
Le service OSRM est opérationnel (Tâche 6) et les positions sont persistées (Tâche 9), mais aucun calcul d'ETA applicatif n'existe.

Étape 0 — Avant de coder
- Confirmer la formule du facteur de correction historique par zone et sa valeur par défaut en l'absence d'historique (proposer une moyenne mobile des écarts ETA OSRM vs temps réel observé, si non déjà précisé dans `SPEC.md`).

TÂCHE
1. Créer un client OSRM encapsulant les appels HTTP (route, distance/durée).
2. Créer un service de calcul ETA combinant la durée OSRM brute et le facteur de correction par zone (valeur par défaut si historique insuffisant).
3. Prévoir le stockage du facteur de correction par zone pour permettre son ajustement ultérieur (Phase 5) — modèle ou champ dédié si non déjà prévu.
4. Gérer les erreurs OSRM (timeout, zone non couverte) en dégradant proprement (ETA indisponible plutôt qu'erreur 500).
5. Ne pas implémenter : le déclenchement des notifications par seuil (Tâche 11).

Contraintes
- L'ETA repose toujours sur OSRM + facteur de correction, jamais sur un map-matching sur un chemin fixe.
- Documenter la précision réelle comme point ouvert à valider en Phase 5.
- Tests avec un mock du client OSRM, pas d'appel réseau réel dans les tests.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme la formule du facteur de correction et sa valeur par défaut.
- En fin de tâche : coche `TODO.md` (Phase 2), documente la formule dans `DECISIONS.md`.

Critère d'acceptation
Le service de calcul ETA retourne une durée corrigée à partir d'une position et d'une destination, en utilisant OSRM et le facteur de correction par zone ; les tests (mock OSRM) passent.

---

## TÂCHE 11 — Logique de seuils, anti-spam et gestion des positions en rafale

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/seuils-notifications-anti-spam`

Constat
Le calcul ETA est disponible (Tâche 10), mais aucune logique de déclenchement de notification n'existe.

Étape 0 — Avant de coder
- Confirmer le modèle de suivi anti-spam à créer (champs, contrainte d'unicité sur (utilisateur, camion_du_jour, seuil, date)) s'il n'existe pas déjà.

TÂCHE
1. Implémenter la logique de seuils (30/20/10/5 min) : à chaque position la plus récente traitée en temps réel (cf. Tâche 9), calculer l'ETA vers chaque utilisateur concerné et déterminer si un seuil est franchi.
2. Implémenter la règle anti-spam : une notification par seuil par (utilisateur, camion_du_jour) par jour.
3. S'assurer que seule la position la plus récente d'une rafale déclenche ce calcul (les positions plus anciennes de la même rafale sont ignorées ici, déjà persistées en Tâche 9).
4. Préparer un point d'extension clair vers l'envoi FCM (Tâche 13), sans encore l'implémenter.
5. Ne pas implémenter : l'envoi FCM réel (Tâche 13), l'auth Firebase (Tâche 12).

Contraintes
- Le citoyen ne reçoit jamais la position exacte du camion — seul le franchissement de seuil est exposé.
- Logique de seuils dans un service dédié, pas dans le listener MQTT ni dans les vues.
- Tests : franchissement de seuil, non-doublon anti-spam le même jour, ignorance des positions non-les-plus-récentes d'une rafale.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le modèle de suivi anti-spam.
- En fin de tâche : coche `TODO.md` (Phase 2).

Critère d'acceptation
Un franchissement de seuil déclenche un événement de notification unique par (utilisateur, camion_du_jour, seuil, jour) ; une rafale de positions historiques ne déclenche qu'un seul calcul ; les tests automatiques passent.

---

## TÂCHE 12 — Intégration Firebase Auth (OTP téléphone) côté backend

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/firebase-auth-backend`

Constat
Le JWT applicatif est configuré (Tâche 4) mais aucune vérification d'identité réelle (OTP téléphone via Firebase) n'existe encore.

Étape 0 — Avant de coder
- Confirmer le flux exact : app Flutter obtient un ID token Firebase après OTP → backend vérifie ce token → backend émet un JWT applicatif.

TÂCHE
1. Configurer le SDK Firebase Admin côté Django pour vérifier les ID tokens Firebase.
2. Créer un endpoint d'échange : ID token Firebase valide → création/récupération de l'`Utilisateur` ou `Chauffeur` correspondant → émission d'un JWT applicatif.
3. Appliquer la règle métier : un compte chauffeur nouvellement créé est en statut `en_attente`, sans accès complet tant que le statut n'est pas `validé`.
4. Gérer les erreurs (token Firebase invalide/expiré) avec des réponses HTTP explicites.
5. Ne pas implémenter : l'écran de validation manuelle chauffeur côté web admin (Phase 4).

Contraintes
- Aucune clé de service Firebase en dur dans le code (variable d'environnement).
- Respecter le statut `en_attente → validé` déjà défini pour les chauffeurs.
- Tests : token Firebase valide (mocké) → JWT émis ; chauffeur `en_attente` → accès restreint conforme à la règle métier.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le flux exact d'échange token Firebase → JWT.
- En fin de tâche : coche `TODO.md` (Phase 2).

Critère d'acceptation
Un ID token Firebase valide permet d'obtenir un JWT applicatif ; un chauffeur `en_attente` ne peut pas accéder aux endpoints réservés aux chauffeurs validés ; les tests automatiques passent.

---

## TÂCHE 13 — Intégration FCM (envoi de notifications)

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md`, `CONVENTIONS.md` et `DECISIONS.md` avant toute modification.

Branche : `feat/fcm-notifications`

Constat
La logique de seuils (Tâche 11) est prête à déclencher des notifications, avec un point d'extension identifié, mais aucun envoi réel n'existe.

Étape 0 — Avant de coder
- Confirmer où sont stockés les tokens FCM des appareils citoyens (champ existant sur `Utilisateur` ou modèle à compléter).

TÂCHE
1. Configurer le SDK Firebase Admin pour l'envoi FCM (peut réutiliser la configuration de la Tâche 12).
2. Implémenter le service d'envoi de notification, branché sur le point d'extension préparé en Tâche 11.
3. Gérer le cas d'un token FCM invalide/expiré (nettoyage ou marquage de l'entrée correspondante) sans faire échouer le traitement du seuil.
4. Ne pas implémenter : la réception/affichage côté app Flutter citoyen (Phase 3).

Contraintes
- Respecter la règle anti-spam déjà en place (Tâche 11) — cette tâche envoie, elle ne re-décide pas si l'envoi doit avoir lieu.
- Tests : envoi mocké (pas d'appel réseau réel FCM), gestion de token invalide testée.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme l'emplacement du stockage des tokens FCM.
- En fin de tâche : coche `TODO.md` (Phase 2) — ce qui clôture la Phase 2 telle que listée actuellement.

Critère d'acceptation
Un franchissement de seuil validé déclenche un envoi FCM réel vers le token de l'utilisateur concerné ; un token invalide est géré sans crash ; les tests automatiques (mock FCM) passent.

---

*Les tâches suivantes (Phase 2 restante si applicable, puis Phase 3 et suivantes) seront rédigées au fur et à mesure, une fois les tâches précédentes validées.*
