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

## TÂCHE 14 — App Chauffeur : auth OTP + écran tournée du jour + start/stop tournée

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §5, §7, §11 et `CONVENTIONS.md` avant toute modification.

Branche : `feat/chauffeur-auth-tournee-du-jour`

Constat
Le flavor chauffeur existe en placeholder (Tâche 3). Le backend expose `/api/auth/token/` (échange Firebase → JWT, Tâche 12), `/api/tournees/du-jour/`, `/api/tournees/{id}/start/` et `/stop/` (SPEC.md §7). Rien n'existe encore côté app au-delà de l'écran placeholder.

Étape 0 — Avant de coder
- Confirmer le flux OTP Firebase côté Flutter (`firebase_auth`, `verifyPhoneNumber`) et le stockage sécurisé du JWT (`flutter_secure_storage`, cf. `CONVENTIONS.md`).

TÂCHE
1. Écran auth OTP dans `lib/driver` : saisie téléphone → code OTP via Firebase Auth → échange de l'ID token contre un JWT applicatif via `/api/auth/token/`.
2. Stockage du JWT (access + refresh) via `flutter_secure_storage`, gestion du refresh automatique.
3. Gestion du statut chauffeur : si `en_attente`, écran d'attente de validation (pas d'accès à la tournée) ; si `valide`, accès à l'écran tournée du jour.
4. Écran tournée du jour : appel à `/api/tournees/du-jour/`, affichage de l'assignation (camion, tournée, zones) ou message "aucune tournée aujourd'hui".
5. Bouton start/stop tournée, appelant `/start/` et `/stop/`, avec état visuel clair (tournée active/inactive).
6. Ne pas implémenter : la capture GPS réelle ni la publication MQTT (Tâche 15), la queue hors-ligne (Tâche 16).

Contraintes
- Architecture en couches (`CONVENTIONS.md`) : logique dans `services/`/`repositories/`, pas dans les widgets.
- Aucun secret Firebase en dur.
- Tests widget minimaux pour l'écran auth et l'écran tournée (mock des appels API).

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme le flux OTP et le mécanisme de stockage du JWT.
- En fin de tâche : coche `TODO.md` (Phase 3).

Critère d'acceptation
Sur le téléphone physique, un chauffeur peut s'authentifier par OTP, voir son statut, consulter sa tournée du jour si validé, démarrer/arrêter la tournée (vérifiable via l'état `AssignationJournaliere` côté backend).

---

## TÂCHE 15 — App Chauffeur : capture GPS + publication MQTT

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §4, §5, `DECISIONS.md` (Tâche 8) et `CONVENTIONS.md` avant toute modification.

Branche : `feat/chauffeur-gps-mqtt`

Constat
Le start/stop tournée existe (Tâche 14) mais ne déclenche encore aucune capture ni publication réelle. Le backend attend des publications sur `camions/{camion_id}/position` (Tâches 8-9). Le point ouvert de `DECISIONS.md` Tâche 8 ("comment le chauffeur récupère les credentials du camion du jour") doit être tranché ici.

Étape 0 — Avant de coder
- Trancher le point ouvert de la Tâche 8 : le backend doit exposer les credentials MQTT du camion assigné (probablement dans la réponse de `/api/tournees/du-jour/` ou `/start/`). Si l'endpoint backend correspondant n'existe pas encore, le signaler avant de coder côté Flutter — un complément backend séparé peut être nécessaire en amont.

TÂCHE
1. Capture GPS (`geolocator` ou équivalent) démarrée uniquement quand la tournée est active (bouton start), stoppée immédiatement au stop — jamais de tracking en tâche de fond hors tournée.
2. Client MQTT (`mqtt_client`) publiant sur `camions/{camion_id}/position`, authentifié avec les credentials récupérés selon le mécanisme confirmé à l'Étape 0.
3. Fréquence adaptative de publication (plus fréquente en mouvement, plus espacée à l'arrêt) — formule à proposer et documenter dans `DECISIONS.md`.
4. Indicateur visuel d'état GPS actif dans l'UI (SPEC.md §11).
5. Ne pas implémenter : la queue locale hors-ligne/retry (Tâche 16) — publication directe uniquement ici.

Contraintes
- QoS 1 ou 2 (SPEC.md).
- Respect strict de la règle "tracking uniquement pendant tournée active".
- Permissions Android runtime (localisation, y compris arrière-plan si nécessaire) gérées proprement, message clair si refusées.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le mécanisme de récupération des credentials MQTT, signale si un complément backend est nécessaire en amont.
- En fin de tâche : coche `TODO.md` (Phase 3), documente la formule de fréquence adaptative dans `DECISIONS.md`.

Critère d'acceptation
Pendant une tournée active sur le téléphone physique, des `PositionCamion` sont créés côté backend à fréquence adaptative ; le tracking s'arrête immédiatement au stop.

---

## TÂCHE 16 — App Chauffeur : queue locale hors-ligne + retry MQTT

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §6 (hors-ligne) et `CONVENTIONS.md` avant toute modification.

Branche : `feat/chauffeur-queue-hors-ligne`

Constat
La publication MQTT directe fonctionne en ligne (Tâche 15), mais toute coupure réseau pendant la tournée ferait perdre les positions capturées.

Étape 0 — Avant de coder
- Confirmer le mécanisme de stockage local (`sqflite` ou file persistée équivalente), cohérent avec la séparation `repositories/` de `CONVENTIONS.md`.

TÂCHE
1. Stocker localement chaque position capturée dès sa capture, indépendamment de la connectivité.
2. Tenter la publication MQTT immédiatement ; en cas d'échec/déconnexion, garder en file d'attente locale et réessayer automatiquement à la reconnexion.
3. Une fois une position confirmée publiée (ack MQTT QoS), la marquer comme envoyée/la purger de la file locale.
4. Indicateur UI "file d'attente hors-ligne en cours d'envoi" (SPEC.md §11) reflétant le nombre de positions en attente.
5. Ne pas implémenter : de nouvelle logique de fréquence adaptative (déjà en Tâche 15) — cette tâche ajoute la robustesse, pas la fréquence.

Contraintes
- Ne jamais perdre de position capturée pendant une tournée active, même après redémarrage de l'app (persistance réelle, pas seulement en mémoire).
- Tests : simuler une coupure réseau, vérifier la mise en file puis l'envoi à la reconnexion.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le mécanisme de stockage local.
- En fin de tâche : coche `TODO.md` (Phase 3).

Critère d'acceptation
En coupant la connectivité du téléphone pendant une tournée active puis en la rétablissant, toutes les positions capturées pendant la coupure finissent par arriver côté backend, dans l'ordre, sans perte.

---

## TÂCHE 17 — App Citoyen : auth OTP + enregistrement du domicile

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §5, §7, §10, §11 et `CONVENTIONS.md` avant toute modification.

Branche : `feat/citoyen-auth-enregistrement-domicile`

Constat
Le flavor citoyen existe en placeholder (Tâche 3). Le backend fournit `/api/auth/token/` et `/api/points-enregistres/` (SPEC.md §7).

Étape 0 — Avant de coder
- Confirmer si l'écran d'auth OTP peut être partagé avec celui du chauffeur (`lib/common`), le flux Firebase étant identique, ou s'il doit rester distinct par flavor pour des raisons de branding (SPEC.md §10, palettes distinctes par app).

TÂCHE
1. Auth OTP citoyen (réutiliser le composant partagé confirmé à l'Étape 0 si applicable), échange token Firebase → JWT.
2. Écran d'enregistrement du domicile : `flutter_map` + recherche d'adresse ou pointage direct sur la carte, création d'un `PointEnregistre` via l'API.
3. Permettre de nommer le point (ex. "Domicile", cohérent avec `PointEnregistre.nom`).
4. Ne pas implémenter : notifications FCM (Tâche 18), écran statut du jour (Tâche 19), carte live (Tâche 22).

Contraintes
- Palette/thème citoyen conforme à `SPEC.md` §10 (vert dominant).
- `flutter_map` + tuiles OSM uniquement, pas de Google Maps.
- Tests widget minimaux pour les deux écrans (mock API).

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le partage ou non de l'écran auth avec le flavor chauffeur.
- En fin de tâche : coche `TODO.md` (Phase 3).

Critère d'acceptation
Sur le téléphone physique, un citoyen peut s'authentifier par OTP et enregistrer un point (domicile) visible ensuite via l'API backend (`GET /api/points-enregistres/`).

---

## TÂCHE 18 — App Citoyen : réception et affichage des notifications FCM

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §7, §9, §11 avant toute modification.

Branche : `feat/citoyen-notifications-fcm`

Constat
Le backend envoie déjà des notifications FCM (Tâche 13), mais rien ne les reçoit/affiche côté app citoyen, et aucun `fcm_token` n'est encore transmis au backend depuis l'app.

Étape 0 — Avant de coder
- Confirmer le point d'intégration pour l'enregistrement du `fcm_token` (endpoint existant ou à compléter côté backend pour mettre à jour `Utilisateur.fcm_token`). Si l'endpoint n'existe pas, le signaler avant de coder — un petit complément backend pourrait être nécessaire en amont, hors périmètre strict de cette tâche mobile.

TÂCHE
1. Configurer `firebase_messaging` côté Flutter (flavor citoyen), récupérer le token FCM de l'appareil.
2. Transmettre/mettre à jour ce token auprès du backend selon le mécanisme confirmé à l'Étape 0.
3. Gérer la réception des notifications en foreground et background (affichage natif Android + gestion du tap pour ouvrir l'app sur l'écran pertinent).
4. Stocker un historique local des notifications reçues (SPEC.md §11).
5. Ne pas implémenter : l'écran statut du jour complet (Tâche 19) au-delà de ce qui est nécessaire pour naviguer depuis une notification.

Contraintes
- Aucune position brute du camion affichée ou stockée côté citoyen — uniquement le contenu de la notification (seuil ETA).
- Tests : simuler une notification reçue (mock) et vérifier son stockage dans l'historique local.

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le mécanisme de transmission du token FCM, signale si un complément backend est nécessaire.
- En fin de tâche : coche `TODO.md` (Phase 3).

Critère d'acceptation
Le token FCM de l'appareil est transmis au backend après connexion ; une notification envoyée depuis le backend (test manuel) est reçue et affichée sur le téléphone physique, foreground et background, et apparaît dans l'historique local.

---

## TÂCHE 19 — App Citoyen : écran statut du jour

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §7, §10, §11 avant toute modification.

Branche : `feat/citoyen-statut-du-jour`

Constat
Le domicile est enregistré (Tâche 17), mais aucun écran n'affiche le calendrier de collecte ni le dernier passage connu. Le backend fournit `GET /api/zones/{id}/calendrier/` (SPEC.md §7).

Étape 0 — Avant de coder
- Confirmer la source du "dernier passage connu" : dérivé du `PositionCamion` le plus récent proche de la zone, ou d'un futur endpoint dédié non encore existant côté backend (à vérifier, pourrait manquer — signaler si c'est le cas).

TÂCHE
1. Écran principal affichant le calendrier de collecte de la zone du domicile enregistré (jours de passage, heure estimée) via l'endpoint calendrier.
2. Affichage du dernier passage connu selon la source confirmée à l'Étape 0.
3. Mise en cache locale du calendrier pour un affichage hors-ligne (stratégie déjà actée : "cache local calendrier" côté citoyen).
4. Code couleur ETA (vert/ambre/rouge) si une notification récente/ETA est disponible, conforme à `SPEC.md` §10 — toujours doublé de texte, jamais la couleur seule.
5. Ne pas implémenter : la carte live (Tâche 22).

Contraintes
- Respect de l'accessibilité définie en `SPEC.md` §10 (taille de police, contraste, pas de couleur seule).
- Tests : affichage correct avec et sans connexion (cache).

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme la source du "dernier passage connu", signale si un complément backend est nécessaire.
- En fin de tâche : coche `TODO.md` (Phase 3).

Critère d'acceptation
L'écran affiche le calendrier de la zone du domicile même hors connexion (cache), et le dernier passage connu quand disponible.

---

## TÂCHE 20 — App Citoyen : formulaire de signalement

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §3, §7, §9 et `CONVENTIONS.md` avant toute modification.

Branche : `feat/citoyen-signalement`

Constat
Aucun écran de signalement n'existe. Le backend fournit `POST /api/signalements/` (SPEC.md §7), avec règle anti-abus (un signalement par utilisateur/camion/type/jour, SPEC.md §9).

Étape 0 — Avant de coder
- Confirmer si le signalement doit être rattaché à un camion précis (si connu) ou seulement à une zone (cas "camion jamais vu") — les deux cas existent dans le modèle `Signalement` (`camion_id`/`zone_id` nullable).

TÂCHE
1. Formulaire de signalement : choix du type (`pas_notifie`/`pas_passe`/`position_incoherente`/`autre`), commentaire optionnel, rattachement automatique à la zone du domicile et au camion du jour si disponible.
2. Appel à `POST /api/signalements/`, gestion de l'erreur de rate limiting/anti-abus (message clair si déjà signalé aujourd'hui pour ce type).
3. Confirmation visuelle claire après envoi réussi.
4. Ne pas implémenter : le tableau de bord des signalements (Phase 4, web admin).

Contraintes
- Respect de `CONVENTIONS.md` (icônes + texte, jamais icône seule pour une action critique).
- Tests : soumission réussie et cas d'erreur anti-abus (mock API).

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le rattachement camion/zone du signalement.
- En fin de tâche : coche `TODO.md` (Phase 3).

Critère d'acceptation
Un signalement soumis depuis le téléphone physique est bien créé côté backend ; une tentative de doublon le même jour pour le même type est rejetée avec un message clair côté app.

---

## TÂCHE 21 — Backend : app `realtime` (Django Channels) — consumer WebSocket + diffusion `position.update`

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md` §5, §6 et `DECISIONS.md` avant toute modification.

Branche : `feat/realtime-websocket-position-update`

Constat
`channels` et `channels_redis` sont dans `requirements.txt` depuis la Tâche 1, mais `CHANNEL_LAYERS`/`ASGI_APPLICATION` n'ont jamais été configurés (écart signalé en Tâche 7, reporté). L'app `realtime` prévue dans `SPEC.md` §5 n'existe pas encore. C'est un prérequis bloquant pour la carte live citoyen (Tâche 22) — cette tâche comble un gap non explicitement séquencé dans `TODO.md` Phase 2/3.

Étape 0 — Avant de coder
- Confirmer le schéma de groupe à utiliser pour le MVP : `camion_<camion_id>` (probable choix le plus simple, cohérent avec un seul camion pilote) plutôt que `zone_<zone_id>` (`SPEC.md` §6 mentionne les deux comme possibles).
- Vérifier si le service `backend` dans `docker-compose.yml` doit passer d'un serveur WSGI à un serveur ASGI (uvicorn/daphne) pour servir les WebSockets, et confirmer l'impact.

TÂCHE
1. Configurer `ASGI_APPLICATION`, `CHANNEL_LAYERS` (`channels_redis`, réutilisant le service `redis` existant).
2. Adapter le service `backend` dans `docker-compose.yml`/`Dockerfile` pour servir l'application via ASGI si nécessaire.
3. Créer l'app `realtime` avec un consumer WebSocket gérant la connexion/déconnexion à un groupe `camion_<camion_id>` (authentification JWT du citoyen requise à la connexion).
4. Brancher la diffusion : dans le service qui traite chaque position (Tâches 9-11), envoyer un événement `position.update` au groupe concerné à chaque position traitée.
5. Fermeture propre de la connexion WebSocket (pas de fuite de connexion si le client ferme l'app sans déconnexion propre).
6. Ne pas implémenter : le client Flutter (Tâche 22) — cette tâche est backend uniquement.

Contraintes
- Le citoyen ne doit recevoir que `camion_id` + `lat`/`lng`/`horodatage` (`SPEC.md` §6), rien d'autre.
- Aucune connexion WebSocket non authentifiée acceptée.
- Tests : test du consumer via le test client Channels (connexion, réception d'un événement `position.update` simulé, déconnexion).

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le schéma de groupe et le changement de serveur ASGI éventuel.
- En fin de tâche : documente dans `GUIDE_DU_DEVELOPPEUR.md` le changement de serveur d'exécution si applicable, ajoute une entrée `DECISIONS.md` sur ce gap comblé, et une case correspondante dans `TODO.md` si absente.

Critère d'acceptation
Un client WebSocket authentifié connecté au groupe d'un camion reçoit un événement `position.update` en temps réel quand une nouvelle position de ce camion est traitée côté backend ; les tests automatiques du consumer passent.

---

## TÂCHE 22 — App Citoyen : carte live optionnelle

Contexte : projet AutoMbalit, dépôt `autombalit-mobile`. Se référer à `SPEC.md` §6, §11 avant toute modification.

Branche : `feat/citoyen-carte-live`

Constat
Le backend expose désormais un WebSocket `position.update` (Tâche 21). Rien n'existe côté app citoyen pour l'afficher.

Étape 0 — Avant de coder
- Confirmer le déclenchement de connexion (uniquement à l'ouverture explicite de l'écran carte, jamais en arrière-plan — `SPEC.md` §6/§11) et la fermeture (à la sortie de l'écran).

TÂCHE
1. Écran carte live avec `flutter_map`, connexion WebSocket au groupe du camion pertinent (celui desservant la zone du domicile, ou celui du dernier passage) uniquement à l'ouverture de l'écran.
2. Mise à jour de la position du marqueur camion en temps réel à réception de `position.update`.
3. Déconnexion automatique et immédiate à la fermeture/sortie de l'écran (pas de connexion persistante en arrière-plan).
4. Gestion de la reconnexion en cas de coupure réseau pendant que l'écran est ouvert.
5. Ne pas implémenter : l'historique de trajet affiché sur la carte (hors périmètre, seule la position courante est affichée).

Contraintes
- Aucune connexion WebSocket tant que l'écran carte n'est pas explicitement ouvert.
- `flutter_map` + OSM uniquement.
- Tests : ouverture/fermeture d'écran déclenchant bien connexion/déconnexion (mock WebSocket).

Process
- Ne fais aucun commit avant "commit".
- Avant de coder : confirme le déclenchement/fermeture de la connexion WebSocket.
- En fin de tâche : coche `TODO.md` (Phase 3) — ce qui clôture la Phase 3 telle que listée actuellement.

Critère d'acceptation
Sur le téléphone physique, ouvrir l'écran carte live affiche la position du camion mise à jour en temps réel pendant une tournée active ; fermer l'écran coupe la connexion WebSocket (vérifiable côté backend, plus de connexion active pour ce client).

---

## TÂCHE 23 — Décision + socle web admin

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md` §5, §8, `CONVENTIONS.md` (§Sécurité, §Partials/Frontend Web Admin) et `DECISIONS.md` avant toute modification.

Branche : `feat/web-admin-socle`

Constat
Aucune interface web admin n'existe. `CONVENTIONS.md` laisse ouvert le choix Django templates classiques vs templates + htmx, à trancher en Phase 4 et consigner dans `DECISIONS.md` avant implémentation. Le style CSS est en revanche déjà tranché : **Tailwind CSS**, via le binaire CLI standalone (pas de dépendance Node/npm obligatoire, cohérent avec la contrainte budget zéro/self-hostable du projet — aucun autre composant du projet ne dépend de Node). Aucun rôle "admin société"/"super admin" n'existe encore au niveau des permissions (seuls `citoyen`/`chauffeur` existent depuis la Tâche 12).

Étape 0 — Avant de coder
- Trancher et documenter le choix templates classiques vs templates + htmx (proposer une recommandation justifiée — probable : htmx pour l'interactivité de la carte Leaflet et des listes filtrables des tâches suivantes, sans architecture SPA lourde).
- Proposer le nom/emplacement de l'app Django dédiée (ex. `web_admin`), cohérent avec la liste d'apps de `SPEC.md` §5.
- Proposer le modèle de permission "admin société" / "super admin" : champ/rôle sur le `User` technique déjà lié 1-1 à `Chauffeur`/`Utilisateur` (Tâche 12), ou nouveau modèle dédié — à confirmer avant de coder.

TÂCHE
1. Créer l'app web admin dédiée, avec un template de base (`base.html`) et une structure de partials par section (upload GeoJSON, calendriers, validation chauffeurs, signalements — un partial par section, aucune logique métier dans les templates, cohérent avec `CONVENTIONS.md`).
2. Intégrer Tailwind CSS via le binaire CLI standalone (pas de `package.json`/Node requis), avec un pipeline de build documenté dans `GUIDE_DU_DEVELOPPEUR.md`.
3. Implémenter l'authentification admin (vue de connexion dédiée, séparée de l'auth Firebase OTP citoyen/chauffeur — un admin utilise email/mot de passe classique Django, cohérent avec `SPEC.md` §11 "Connexion admin société").
4. Implémenter le modèle de permission confirmé à l'Étape 0 : un "admin société" ne voit/gère que les données de sa société, un "super admin" voit tout — appliqué de façon réutilisable (mixin/decorator) pour toutes les vues web admin à venir, et cohérent avec la règle `CONVENTIONS.md` sur `/api/admin/...`.
5. Ne pas implémenter : les fonctionnalités métier elles-mêmes (upload GeoJSON, calendriers, validation chauffeurs, signalements) — uniquement le socle. Pages de ces sections en placeholder simple ("à venir").

Contraintes
- Pas de dépendance à un SDK cartographique payant (Leaflet + OSM, cohérent avec le reste du projet) — pas encore utilisé dans cette tâche, mais à garder en tête pour la Tâche 25.
- Respect strict de l'architecture en couches (`CONVENTIONS.md`) : vues fines, logique dans des services dédiés dès qu'elle dépasse une simple requête.
- Tests : accès refusé sans authentification, accès refusé à un rôle non autorisé, accès correct par rôle.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme le choix templates vs htmx, le nom de l'app, et le modèle de permission.
- En fin de tâche : coche `TODO.md` (Phase 4, "trancher le choix frontend"), documente les décisions (templates/htmx, Tailwind, modèle de permission) dans `DECISIONS.md`.

Critère d'acceptation
Un compte admin société peut se connecter et accède à un tableau de bord de base (liens vers les sections à venir) ; un compte citoyen/chauffeur ne peut pas accéder à cette interface ; un admin société d'une autre société ne voit pas les données d'une société qui n'est pas la sienne (vérifiable dès qu'une première donnée scoped existera, sinon test de principe sur le mixin de permission).

---

## TÂCHE 24 — Validation des comptes chauffeurs (web admin)

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md` §3, §9, §11 et `DECISIONS.md` avant toute modification.

Branche : `feat/web-admin-validation-chauffeurs`

Constat
Le champ `Chauffeur.statut_validation` (`en_attente`/`valide`/`rejete`) existe depuis la Tâche 2, mais aucune interface ne permet de le faire évoluer — jusqu'ici uniquement fait manuellement en base via des commandes de test (`reset_tournee_test` et fixtures). C'est le seul point réellement bloquant pour un usage terrain sans intervention manuelle en base de données.

Étape 0 — Avant de coder
- Vérifie si la distribution des credentials MQTT au chauffeur nouvellement validé est déjà entièrement gérée côté backend (Tâche 8/15 : credentials exposés via `/api/tournees/du-jour/` ou `/start/`) — si un point reste ouvert à ce sujet, signale-le avant de continuer, sans le traiter dans cette tâche si ça sort du périmètre "validation".

TÂCHE
1. Liste des chauffeurs `en_attente` pour la société de l'admin connecté (toutes sociétés si super admin), avec les informations utiles à la décision (téléphone, date de création de compte).
2. Actions "Valider" / "Rejeter" sur chaque chauffeur, mettant à jour `statut_validation`.
3. Vue liste secondaire des chauffeurs déjà validés/rejetés (historique, pas d'action dessus dans cette tâche).
4. Ne pas implémenter : la création manuelle d'un chauffeur depuis le web admin (un chauffeur crée son compte via l'app avec Firebase Auth, cf. Tâche 12) — cette tâche ne fait que faire évoluer un statut existant.

Contraintes
- Un admin société ne peut valider/rejeter que les chauffeurs de sa propre société (`societe` FK) — un super admin peut agir sur tous.
- Toute action de validation/rejet doit être auditable a minima (log applicatif, pas nécessairement un modèle d'historique dédié dans cette tâche — à toi de juger si c'est nécessaire, sinon le signaler).
- Tests : validation/rejet appliqués correctement, admin société bloqué sur les chauffeurs d'une autre société.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme le résultat de la vérification de l'Étape 0.
- En fin de tâche : coche `TODO.md` (Phase 4).

Critère d'acceptation
Un admin société peut valider ou rejeter un chauffeur `en_attente` de sa société ; le changement de statut est immédiatement reflété côté API (`statut_validation`, vérifiable via l'app chauffeur ou l'admin Django natif) ; un admin société ne peut pas agir sur un chauffeur d'une autre société.

---

## TÂCHE 25 — Interface upload GeoJSON + visualisation carte (Leaflet)

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md` §7, §9, §11 et `DECISIONS.md` (stratégie d'import GeoJSON actée en Tâche 5) avant toute modification.

Branche : `feat/web-admin-upload-geojson-carte`

Constat
L'endpoint d'upload et de validation GeoJSON existe côté API depuis la Tâche 5 (SRID 4326, type Polygon, properties requises, stratégie all-or-nothing). Aucune interface web ne permet à un admin de l'utiliser — jusqu'ici uniquement testé via des commandes de management ou l'endpoint API directement.

Étape 0 — Avant de coder
- Confirme si cette interface appelle l'endpoint API existant (Tâche 5) en HTTP interne, ou réutilise directement le service de validation en Python depuis la vue web — à trancher selon ce qui est le plus cohérent avec l'architecture en couches déjà en place (probable : réutiliser le service directement, pas un aller-retour HTTP interne).

TÂCHE
1. Formulaire d'upload d'un fichier GeoJSON, scoped à la société de l'admin connecté.
2. Affichage clair des erreurs de validation (SRID, géométrie, properties) retournées par le service existant, feature par feature si plusieurs erreurs.
3. Carte Leaflet (tuiles OSM, pas de dépendance payante) affichant les zones existantes de la société de l'admin, avec un rafraîchissement après un upload réussi.
4. Ne pas implémenter : l'édition d'une zone existante après upload (hors périmètre), la gestion des tournées elle-mêmes (déjà couverte par l'API CRUD de la Tâche 4, pas re-fait ici en UI web sauf si tu juges que c'est nécessaire pour rendre l'écran utile — dans ce cas, signale-le avant de l'ajouter).

Contraintes
- Pas de dépendance à un SDK cartographique payant (Leaflet + OSM uniquement).
- Respect de la stratégie d'import déjà actée (Tâche 5, `DECISIONS.md`) — ne pas la redéfinir.
- Tests : upload valide crée la zone et l'affiche sur la carte, upload invalide affiche les erreurs sans créer de zone.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme le mécanisme d'appel au service de validation existant.
- En fin de tâche : coche `TODO.md` (Phase 4).

Critère d'acceptation
Un admin société peut uploader un GeoJSON valide et voir la nouvelle zone apparaître sur la carte Leaflet ; un GeoJSON invalide affiche un message d'erreur clair sans créer de zone.

---

## TÂCHE 26 — Gestion des calendriers de collecte par zone

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md` §3, §7, §11 avant toute modification.

Branche : `feat/web-admin-calendriers`

Constat
`GET /api/zones/{id}/calendrier/` existe côté API (complément découvert pendant la Tâche 19), mais aucun endpoint d'écriture ni interface web ne permet de créer/modifier un `CalendrierCollecte` — jusqu'ici uniquement fait via une commande de management de test.

Étape 0 — Avant de coder
- Vérifie si un endpoint API d'écriture sur `CalendrierCollecte` existe déjà ; si non, ce sera un complément backend nécessaire avant l'interface web, comme pour les tâches précédentes ayant révélé un manque similaire (14, 15, 17, 18).

TÂCHE
1. Si nécessaire (cf. Étape 0), ajoute un endpoint CRUD minimal sur `CalendrierCollecte`, scoped à la société de l'utilisateur (via la `Zone` associée).
2. Liste des calendriers par zone pour la société de l'admin connecté.
3. Formulaire de création/édition (jour de la semaine, heure estimée, tournée associée).
4. Suppression d'une entrée de calendrier.
5. Ne pas implémenter : la génération automatique de calendrier à partir de l'historique de positions (hors périmètre, resterait manuel pour l'instant).

Contraintes
- Un admin société ne gère que les calendriers des zones desservies par des tournées de sa société.
- Tests : création/édition/suppression fonctionnelles, scoping par société respecté.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme le résultat de la vérification de l'Étape 0 (endpoint d'écriture à créer ou déjà existant).
- En fin de tâche : coche `TODO.md` (Phase 4).

Critère d'acceptation
Un admin société peut créer, modifier et supprimer une entrée de calendrier pour une zone de sa société ; le résultat est immédiatement visible via `GET /api/zones/{id}/calendrier/` (vérifiable côté app citoyen, écran statut du jour).

---

## TÂCHE 27 — Tableau de bord des signalements par zone/société

Contexte : projet AutoMbalit, dépôt `autombalit-backend`. Se référer à `SPEC.md` §3, §9, §11 avant toute modification.

Branche : `feat/web-admin-signalements`

Constat
Le modèle `Signalement` existe et l'app citoyen peut en créer (Tâche 20), mais aucune vue web admin ne permet à une société de les consulter — dernière tâche de la Phase 4, peu prioritaire tant qu'aucun vrai citoyen n'utilise l'app en dehors des tests.

Étape 0 — Avant de coder
- Vérifie les champs exacts du modèle `Signalement` (type, commentaire, zone/camion nullable, horodatage) et confirme s'il existe déjà un champ de statut de traitement ("traité"/"non traité") — sinon, propose s'il faut l'ajouter dans cette tâche ou la laisser en lecture seule pour l'instant.

TÂCHE
1. Liste des signalements pour la société de l'admin connecté, filtrable par zone, type et période.
2. Vue agrégée simple (nombre de signalements par zone, par type) pour repérer les zones à problème.
3. Si un champ de statut de traitement est confirmé nécessaire à l'Étape 0, ajoute-le avec une action "marquer comme traité" — sinon, cette tâche reste en lecture seule/consultation.
4. Ne pas implémenter : de réponse automatique au citoyen, ni de lien avec l'ajustement du facteur de correction ETA (mentionné comme piste en `SPEC.md` §8, mais hors périmètre MVP).

Contraintes
- Un admin société ne voit que les signalements liés à sa société (via la zone/tournée desservie).
- Tests : filtrage correct, scoping par société respecté, agrégation correcte sur un petit jeu de données de test.

Process
- Ne fais aucun commit avant que je te dise explicitement "commit".
- Avant de coder : confirme la nécessité ou non d'un champ de statut de traitement.
- En fin de tâche : coche `TODO.md` (Phase 4) — ce qui clôture la Phase 4 telle que listée actuellement.

Critère d'acceptation
Un admin société voit la liste et l'agrégation des signalements de sa société, filtrable par zone/type/période ; un admin d'une autre société ne voit pas ces données.

---

*Les tâches suivantes (Phase 5 — test pilote réel) seront rédigées au fur et à mesure, une fois les tâches précédentes validées.*
