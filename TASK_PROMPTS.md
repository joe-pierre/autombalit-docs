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

*Les tâches suivantes (Phase 1 étapes 3-6, puis Phase 2 et suivantes) seront rédigées au fur et à mesure, une fois les tâches précédentes validées.*
