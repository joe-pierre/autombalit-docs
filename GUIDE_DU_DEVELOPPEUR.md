# 🚚 Guide du Développeur — AutoMbalit
> Suivre dans l'ordre, ne rien sauter. Ce guide couvre le setup local, le déploiement VPS complet, et l'utilisation finale de l'application par les trois profils (chauffeur, citoyen, admin).
> Stack de référence : voir `SPEC.md` §2. Toute déviation doit être actée dans `DECISIONS.md`.

---

## VUE D'ENSEMBLE

```
PHASE 0  — Prérequis locaux         : outils à installer sur ta machine de dev
PHASE 1  — Backend local            : Django + PostGIS + Redis + Mosquitto + OSRM en local
PHASE 2  — Mobile local             : Flutter (apps chauffeur + citoyen) en local
PHASE 3  — Infrastructure VPS       : serveur + domaine + DNS
PHASE 4  — Serveur VPS              : OS + stack système (Python, PostGIS, Redis, Mosquitto, OSRM, Nginx)
PHASE 5  — Code en production       : clone, dépendances, .env, migrations
PHASE 6  — Services production      : Supervisor (ASGI, MQTT listener, Celery), Nginx, SSL
PHASE 7  — Vérification             : tests de bout en bout, logs
PHASE 8  — Maintenance              : script de déploiement, mises à jour
PHASE 9  — Publication mobile       : build & signature Android, Play Store
PHASE 10 — Utilisation finale       : parcours complet des 3 profils utilisateurs
```

---

## PHASE 0 — PRÉREQUIS LOCAUX

Environnement de dev basé sur **Docker Compose** : PostGIS, Redis, Mosquitto et OSRM tournent en conteneurs, un seul point d'entrée (`docker compose up`), pas d'installation manuelle de services système sur ta machine. Le test mobile se fait sur un **appareil Android physique (Samsung) branché en USB**, pas sur un émulateur.

Outils à installer une fois sur ta machine de développement :

```bash
# Docker + Docker Compose (déjà présent sur ta machine)
docker --version
docker compose version

# Python 3.12+ (utile pour lancer des commandes manage.py ponctuelles hors conteneur, ex. génération de migrations)
python3 --version

# Flutter SDK
flutter --version
flutter doctor   # doit être vert sur Android toolchain au minimum

# Git
git --version

# ADB (Android Debug Bridge) — généralement inclus avec le SDK Android installé par Flutter
adb version
```

Compte Firebase à créer (gratuit) sur https://console.firebase.google.com — un seul projet Firebase pour Auth + FCM, partagé entre l'app chauffeur et l'app citoyen.

### Préparer le téléphone Samsung pour le développement

1. Sur le téléphone : **Paramètres → À propos du téléphone → Numéro de build** → appuyer 7 fois pour activer le mode développeur.
2. **Paramètres → Options pour développeurs** → activer **Débogage USB**.
3. Brancher le téléphone à la machine de dev via câble USB.
4. Autoriser l'ordinateur sur la popup qui apparaît sur le téléphone (« Autoriser le débogage USB ? »).
5. Vérifier la détection :

```bash
adb devices
# doit lister le téléphone avec le statut "device" (pas "unauthorized")
```

Si le statut est `unauthorized`, débrancher/rebrancher le câble et réautoriser sur le téléphone.

---

## PHASE 1 — BACKEND LOCAL (DOCKER COMPOSE)

### 1.1 Créer le dossier du projet et initialiser Git

Rien n'existe encore — on part de zéro en local.

```bash
mkdir autombalit-backend && cd autombalit-backend
git init
git branch -M main
```

### 1.2 Créer le dépôt distant sur GitHub

**Option A — via l'interface web**
1. Aller sur https://github.com/new
2. Nom : `autombalit-backend`
3. Visibilité : privé (recommandé pour l'instant)
4. Ne **pas** cocher "Add a README" (on le fait en local juste après)
5. Créer → noter l'URL affichée (HTTPS ou SSH selon ta config)

**Option B — via GitHub CLI** (si `gh` est installé : `gh --version`)

```bash
gh auth login   # une seule fois si pas déjà fait
gh repo create autombalit-backend --private --source=. --remote=origin
```

Si Option A, relier le dépôt distant manuellement :

```bash
git remote add origin git@github.com:TON_COMPTE/autombalit-backend.git
```

### 1.3 Premier commit et scaffolding Django

```bash
echo "# AutoMbalit — Backend" > README.md

cat > .gitignore << 'EOF'
venv/
__pycache__/
*.pyc
.env
.env.*
!.env.example
firebase-service-account.json
staticfiles/
media/
osrm-data/
db.sqlite3
EOF

git add README.md .gitignore
git commit -m "chore: initialisation du dépôt"
git push -u origin main
```

> Le `!` de `!.env.example` déclenche l'expansion d'historique en zsh/bash si on l'écrit dans un `echo "..."` classique — ça casse la commande avec une erreur `event not found`. Le heredoc `<< 'EOF'` (délimiteur entre guillemets simples) évite ce piège, aussi bien pour ce fichier que pour tout futur fichier contenant un `!`, un `$`, ou des backticks.

Créer l'environnement virtuel local (nécessaire pour générer le squelette Django avant de tout faire tourner en Docker) :

```bash
python3 -m venv venv
source venv/bin/activate      # Windows : venv\Scripts\activate

pip install django djangorestframework djangorestframework-gis \
  channels channels-redis djangorestframework-simplejwt \
  paho-mqtt firebase-admin psycopg2-binary

django-admin startproject autombalit_backend .
```

Créer les apps définies dans `SPEC.md` §5 :

```bash
python manage.py startapp core
python manage.py startapp tracking
python manage.py startapp citizens
python manage.py startapp notifications
python manage.py startapp realtime
python manage.py startapp geo_import
```

Figer les dépendances :

```bash
pip freeze > requirements.txt
```

Commit intermédiaire :

```bash
git add .
git commit -m "chore: scaffolding Django + apps"
git push
```

> À partir d'ici, le contenu réel de chaque app (modèles, services, serializers — voir `CONVENTIONS.md` §Architecture et paradigmes) est à développer via les prompts de `TASK_PROMPTS.md` donnés à Claude Code, sur une branche dédiée par tâche, plutôt qu'en écrivant le code à la main ligne par ligne.

### 1.4 Créer le fichier `docker-compose.yml`

```bash
nano docker-compose.yml
```

```yaml
services:
  db:
    image: postgis/postgis:16-3.4
    restart: unless-stopped
    environment:
      POSTGRES_DB: autombalit_dev
      POSTGRES_USER: autombalit_user
      POSTGRES_PASSWORD: mot_de_passe_dev
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    ports:
      - "6379:6379"

  mosquitto:
    image: eclipse-mosquitto:2
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - ./docker/mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./docker/mosquitto/passwd:/mosquitto/config/passwd
      - ./docker/mosquitto/acl:/mosquitto/config/acl

  osrm:
    image: osrm/osrm-backend
    restart: unless-stopped
    command: osrm-routed --algorithm mld /data/senegal-and-gambia-latest.osrm
    volumes:
      - ./osrm-data:/data
    ports:
      - "5000:5000"

  backend:
    build: .
    restart: unless-stopped
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      - db
      - redis
      - mosquitto
      - osrm

  mqtt_listener:
    build: .
    restart: unless-stopped
    command: python manage.py run_mqtt_listener
    volumes:
      - .:/app
    env_file: .env
    depends_on:
      - db
      - mosquitto

volumes:
  db_data:
```

Config Mosquitto pour le dev — authentification et ACL activées dès le local, pas seulement en production (voir §1.4bis juste après) :

```bash
mkdir -p docker/mosquitto
nano docker/mosquitto/mosquitto.conf
```

```conf
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl
```

> Avant le tout premier `docker compose up`, créer les fichiers vides attendus par ces deux chemins (sinon Docker monte un dossier à la place d'un fichier) : `touch docker/mosquitto/passwd docker/mosquitto/acl`. Ni l'un ni l'autre n'est commité (`.gitignore`) — voir §1.4bis pour les provisionner.

### 1.4bis — Authentification et ACL Mosquitto (comptes MQTT par camion)

**Schéma de topics** : `camions/{camion_id}/position` (voir `CONVENTIONS.md`), inchangé.

**Credentials liés au camion, pas au chauffeur** : l'assignation chauffeur/camion est journalière (`AssignationJournaliere`), donc un chauffeur n'a pas de "topic à lui" fixe. Chaque **camion** a son propre compte MQTT (`camion_<camion_id>`), autorisé à publier uniquement sur `camions/<camion_id>/position` — jamais à lire quoi que ce soit. Le compte backend (`autombalit_backend`, identifiants dans `.env`) a lui un accès en lecture seule à `camions/#` (nécessaire au futur `mqtt_listener`, Tâche 9). Voir `DECISIONS.md` pour la justification complète de ce choix et son point ouvert (comment le chauffeur récupère les credentials du camion qui lui est assigné le jour même — hors périmètre de la Tâche 8).

**Provisionner le compte backend** (une fois, après avoir renseigné `MQTT_USERNAME`/`MQTT_PASSWORD` dans `.env`) :

```bash
docker compose up -d mosquitto
bash scripts/provision_mqtt_backend.sh
```

**Provisionner un camion** (à chaque ajout d'un camion à la flotte) :

```bash
bash scripts/provision_mqtt_camion.sh <camion_id>
# ou avec un mot de passe imposé plutôt que généré :
bash scripts/provision_mqtt_camion.sh <camion_id> <mot_de_passe>
```

Le mot de passe (généré ou fourni) n'est affiché qu'une fois par le script, jamais stocké en clair — à transmettre par un canal sécurisé jusqu'à ce qu'un mécanisme de distribution automatisé au chauffeur du jour existe (tâche ultérieure).

**Vérifier l'ACL** (deux camions de test provisionnés automatiquement, nettoyés en fin de script) :

```bash
bash scripts/test_mqtt_acl.sh
```

Confirme qu'un camion publie bien sur son propre topic et que sa tentative sur le topic d'un autre camion n'est jamais relayée au backend.

> **Avertissements de permissions au démarrage de Mosquitto** (`world readable permissions`, `owner is not mosquitto`) : attendus et non bloquants en dev local — les fichiers `passwd`/`acl` sont montés depuis l'hôte (UID différent de l'utilisateur `mosquitto` du conteneur), contrairement à la Phase 4.4 (VPS) où ces fichiers appartiennent nativement à root avec des permissions strictes. Ne pas `chmod 600`/`700` ces fichiers en local : le conteneur ne pourrait alors plus les lire (UID hôte ≠ UID conteneur) et Mosquitto s'arrêterait au prochain rechargement.
>
> **Recharger la config après provisioning** : un `SIGHUP` fait en réalité terminer le process Mosquitto de cette image plutôt que recharger proprement `passwd`/`acl` à chaud — les deux scripts utilisent donc `docker compose restart mosquitto` (repris automatiquement grâce à `restart: unless-stopped`), plus prévisible qu'un signal.

### 1.5 Fichier `.env` local

Le fichier `.env.example` n'existe pas encore — on part d'un projet vide, pas d'un clone. Le créer d'abord (c'est ce fichier, sans secrets réels, qui sera commité) :

```bash
cat > .env.example << 'EOF'
DEBUG=True
SECRET_KEY=change-me
DATABASE_URL=postgis://autombalit_user:change-me@db:5432/autombalit_dev

REDIS_URL=redis://redis:6379/0

MQTT_BROKER_HOST=mosquitto
MQTT_BROKER_PORT=1883
MQTT_USERNAME=
MQTT_PASSWORD=

OSRM_BASE_URL=http://osrm:5000

FIREBASE_CREDENTIALS_JSON=./firebase-service-account.json
FCM_PROJECT_ID=change-me
EOF

git add .env.example
git commit -m "chore: ajout de .env.example"
git push
```

Puis créer le `.env` réel (jamais commité, déjà exclu par `.gitignore`) à partir de ce modèle :

```bash
cp .env.example .env
nano .env
```

Remplacer les valeurs de `.env` par les vraies valeurs de dev :

```env
DEBUG=True
SECRET_KEY=une-cle-de-dev-quelconque
DATABASE_URL=postgis://autombalit_user:mot_de_passe_dev@db:5432/autombalit_dev

REDIS_URL=redis://redis:6379/0

MQTT_BROKER_HOST=mosquitto
MQTT_BROKER_PORT=1883
MQTT_USERNAME=
MQTT_PASSWORD=

OSRM_BASE_URL=http://osrm:5000

FIREBASE_CREDENTIALS_JSON=./firebase-service-account.json
FCM_PROJECT_ID=ton-projet-firebase
```

> Noter la différence avec un `.env` hors Docker : les hôtes (`db`, `redis`, `mosquitto`, `osrm`) sont les noms des services Docker Compose, pas `localhost`, car chaque conteneur se résout par son nom de service sur le réseau Docker interne.

Télécharger la clé de service Firebase (Console Firebase → Paramètres du projet → Comptes de service → Générer une nouvelle clé privée) et la placer à la racine sous `firebase-service-account.json` (déjà dans `.gitignore`, ne jamais committer ce fichier).

### 1.6 Préparer les données OSRM (une fois, avant le premier `docker compose up`)

```bash
mkdir -p osrm-data && cd osrm-data
curl -fLO https://download.geofabrik.de/africa/senegal-and-gambia-latest.osm.pbf

docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/senegal-and-gambia-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/senegal-and-gambia-latest.osrm
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/senegal-and-gambia-latest.osrm
cd ..
```

Ces trois commandes ne sont à relancer que si l'extrait OSM change — pas à chaque démarrage.

> **Alternative scriptée** : `bash scripts/prepare_osrm_data.sh` (à la racine de `autombalit-backend/`) exécute ces mêmes étapes (téléchargement + extract/partition/customize) en une seule commande, idempotent sur le téléchargement (ne retélécharge pas si le `.osm.pbf` est déjà présent). À utiliser pour régénérer les données après une mise à jour de l'extrait Geofabrik. Une fois la régénération terminée, redémarrer le service : `docker compose restart osrm`.
>
> **Périmètre retenu** : extrait Sénégal+Gambie complet (pas de découpage par région), le quartier pilote n'étant pas encore choisi (`TODO.md` Phase 0) — voir `DECISIONS.md`. **Profil de routing** : `car.lua` par défaut (fourni par l'image `osrm/osrm-backend`) — approximation « camion ≈ voiture » assumée en attendant une validation terrain (Phase 5), voir `DECISIONS.md`.
>
> **Vérifier que le service répond correctement** une fois `docker compose up` lancé :
> ```bash
> curl "http://localhost:5000/route/v1/driving/-17.4467,14.6928;-17.44,14.70"
> ```
> Doit retourner un JSON `"code":"Ok"` avec une route calculée (distance/durée) — identique au test de la section 1.10 ci-dessous.

### 1.7 Écrire le `Dockerfile` du backend (si absent)

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y \
    gdal-bin libgdal-dev gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
```

> `gdal-bin`/`libgdal-dev` sont nécessaires pour GeoDjango.

### 1.8 Lancer tout l'environnement

```bash
docker compose up --build
```

`db`, `redis`, `mosquitto` et `osrm` exposent chacun un `healthcheck` (`pg_isready`, `redis-cli ping`, test TCP via `nc`, test TCP via `bash -c "</dev/tcp/..."` — ces deux dernières images n'embarquent ni `curl` ni `wget`). `backend` et `mqtt_listener` déclarent `depends_on: condition: service_healthy` sur ces quatre services : ils ne démarrent qu'une fois que `db`/`redis`/`mosquitto`/`osrm` sont effectivement prêts à accepter des connexions, pas seulement une fois leur conteneur lancé — ça évite une race condition au premier démarrage (ex. `backend` qui tenterait de se connecter à PostGIS avant que Postgres accepte des connexions). `backend` a lui-même un `healthcheck` (test TCP sur le port 8000, pas d'endpoint `/health/` dédié à ce stade).

> **Attendu à ce stade** : le service `mqtt_listener` va boucler en erreur (`Unknown command: 'run_mqtt_listener'`), et donc rester en `Restarting` même une fois `db`/`mosquitto` `healthy`. C'est normal — cette commande Django n'existe pas encore, elle sera créée lors de la tâche de développement MQTT/ETA (Phase 2, avec Claude Code). En attendant, `docker compose stop mqtt_listener` évite de polluer les logs ; les autres services (`db`, `redis`, `mosquitto`, `osrm`, `backend`) doivent, eux, démarrer et passer `(healthy)` sans erreur.

**Ordre de vérification en cas de problème au démarrage** :
1. `docker compose ps` — repérer quel service n'est pas `(healthy)` (ou reste `starting`/`unhealthy`). Ordre de dépendance : `db`, `redis`, `mosquitto`, `osrm` d'abord (aucune dépendance entre eux), puis `backend`/`mqtt_listener` seulement une fois les quatre premiers `healthy`.
2. `docker compose logs <service>` sur le service en cause, en commençant toujours par la dépendance la plus en amont qui ne serait pas saine (inutile de déboguer `backend` si `db` n'est pas encore `healthy`).
3. Si un service reste bloqué en `starting` au-delà du nombre de tentatives (`retries` du healthcheck), c'est le signe d'un vrai problème (pas juste une lenteur de démarrage) — vérifier la configuration (`.env`, volumes montés) plutôt que d'augmenter `retries` par réflexe.

> **Note sur les logs `backend`/`mqtt_listener`** : le `Dockerfile` fixe `ENV PYTHONUNBUFFERED=1`. Sans ça, la sortie standard de Django est bufferisée dans un conteneur (pas de TTY détecté) et `docker compose logs` peut sembler figé (seul `Watching for file changes...` s'affiche) alors que le serveur tourne normalement — ce n'était qu'un problème d'affichage des logs, pas un blocage réel.

> **Écart connu, assumé** : `redis`, `channels` et `channels_redis` sont dans `requirements.txt` depuis la Tâche 1, et le service `redis` est bien accessible depuis `backend` (testé via `redis.Redis.from_url(...).ping()`), mais `settings.py` ne déclare encore ni `CHANNEL_LAYERS` ni `ASGI_APPLICATION` — Django n'utilise pas encore Redis. Le câblage de Channels est prévu en Phase 2 (temps réel citoyen, voir `SPEC.md` §6), volontairement non fait dans la Tâche 7 dont le périmètre est l'orchestration Docker Compose, pas l'implémentation applicative.

Dans un autre terminal, exécuter les migrations et créer un superutilisateur :

```bash
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py createsuperuser
```

Vérifier : http://localhost:8000/admin doit être accessible depuis le navigateur de la machine hôte.

### 1.9 Tester le flux MQTT manuellement

```bash
# S'abonner comme le ferait le backend
docker compose exec mosquitto mosquitto_sub -h localhost -t "camions/+/position"

# Dans un autre terminal, simuler une position chauffeur
docker compose exec mosquitto mosquitto_pub -h localhost -t "camions/test123/position" \
  -m '{"lat": 14.6928, "lng": -17.4467, "horodatage": "2026-09-10T12:00:00Z"}'
```

Le message doit apparaître côté `mosquitto_sub`, puis dans les logs du service `mqtt_listener` (`docker compose logs -f mqtt_listener`).

### 1.10 Tester OSRM

```bash
curl "http://localhost:5000/route/v1/driving/-17.4467,14.6928;-17.44,14.70"
```

Doit retourner un JSON avec une route calculée.

### 1.11 Commandes Docker Compose utiles au quotidien

```bash
docker compose up -d              # démarrer tout en arrière-plan
docker compose down               # tout arrêter (les données restent dans le volume db_data)
docker compose logs -f backend    # suivre les logs d'un service en particulier
docker compose exec backend python manage.py <commande>   # exécuter une commande Django dans le conteneur
docker compose exec backend bash  # ouvrir un shell dans le conteneur backend
docker compose build backend      # reconstruire l'image après modification de requirements.txt
docker compose exec backend pytest              # lancer la suite de tests (pytest-django + factory_boy, voir CONVENTIONS.md §Tests)
docker compose exec backend pytest -v chemin/vers/test_xxx.py   # cibler un fichier de test précis, en mode verbeux
```

Avec Docker Compose, un seul terminal (en mode `-d` ou détaché) suffit pour faire tourner l'ensemble de la stack locale — backend, base, Redis, Mosquitto, OSRM et le listener MQTT.

---

## PHASE 2 — MOBILE LOCAL (FLUTTER SUR TÉLÉPHONE PHYSIQUE)

### 2.1 Générer le projet Flutter (crée aussi le dossier)

```bash
flutter create --org com.autombalit --project-name autombalit_mobile autombalit-mobile
cd autombalit-mobile
```

> Le nom de dossier (`autombalit-mobile`, avec tiret, cohérent avec le nom du dépôt GitHub) et le nom de package Dart (`autombalit_mobile`, avec underscore — obligatoire, un package Dart ne peut pas contenir de tiret) sont volontairement différents. `--project-name` permet cette distinction en une seule commande, sans `mkdir` séparé.

### 2.2 Initialiser Git

`flutter create` ne lance pas `git init` tout seul — mais le `.gitignore` qu'il génère (`build/`, `.dart_tool/`, etc.) est déjà en place, donc autant initialiser Git maintenant pour en profiter dès le premier commit :

```bash
git init
git branch -M main
```

### 2.3 Créer le dépôt distant sur GitHub

Même procédure qu'en Phase 1.2 (interface web ou `gh repo create`), avec le nom `autombalit-mobile`.

```bash
gh repo create autombalit-mobile --private --source=. --remote=origin
# ou, après création manuelle sur github.com :
git remote add origin git@github.com:TON_COMPTE/autombalit-mobile.git
```

Vérifier que ça compile à vide :

```bash
flutter pub get
flutter analyze
```

Premier commit :

```bash
git add .
git commit -m "chore: scaffolding Flutter initial"
git push -u origin main
```

> La configuration des flavors (`citoyen`/`chauffeur`) et de la structure de dossiers (`lib/common/`, `lib/driver/`, `lib/citizen/`, `lib/shared_widgets/`) n'est pas faite manuellement à ce stade — voir `TASK_PROMPTS.md` — Tâche 3, à donner à Claude Code une fois ce commit initial poussé. Les commandes `flutter run --flavor ...` des sections suivantes supposent cette tâche déjà réalisée.

### 2.4 Configurer Firebase côté Flutter

```bash
dart pub global activate flutterfire_cli
flutterfire configure
```

Suivre les invites : sélectionner le projet Firebase créé en Phase 0, cocher Android (et iOS plus tard). Ça génère `firebase_options.dart` et `google-services.json` — ne pas les committer s'ils contiennent des identifiants sensibles (vérifier `.gitignore`).

### 2.5 Vérifier que le téléphone est bien détecté

Câble branché, débogage USB activé et autorisé (voir Phase 0) :

```bash
flutter devices
```

Le Samsung doit apparaître dans la liste (ex. `SM-A155F (mobile) • adb-xxxxx • android-arm64`). S'il n'apparaît pas :
- Vérifier `adb devices` directement — si le téléphone n'y figure pas non plus, tester un autre câble (certains câbles USB sont "charge only", sans transfert de données) ou un autre port.
- Sur certains Samsung, une popup "Autoriser le transfert de fichiers/débogage" doit être validée à chaque nouvelle connexion à une machine — vérifier l'écran du téléphone.

### 2.6 Configurer l'URL de l'API locale accessible depuis le téléphone

Le téléphone physique n'est **pas** sur le même hôte que la machine de dev (contrairement à un émulateur qui peut utiliser `10.0.2.2`). Deux options :

**Option A — même réseau Wi-Fi que la machine de dev (recommandé)**

Récupérer l'IP locale de la machine :

```bash
# macOS/Linux
ip addr show | grep "inet " # ou : ifconfig | grep "inet "
# Windows
ipconfig
```

Dans la config d'environnement de l'app (`lib/common/config/env.dart` ou équivalent) :

```dart
const String apiBaseUrl = "http://192.168.x.x:8000/api";  // IP locale de la machine de dev
const String wsBaseUrl = "ws://192.168.x.x:8000/ws";
```

Le port 8000 doit être exposé par `docker compose` (déjà le cas dans le `docker-compose.yml` de la Phase 1) et le pare-feu de la machine de dev ne doit pas bloquer les connexions entrantes sur ce port depuis le réseau local.

**Option B — via le câble USB avec `adb reverse` (fonctionne aussi sans Wi-Fi partagé)**

```bash
adb reverse tcp:8000 tcp:8000
```

Cette commande redirige le port 8000 du téléphone vers le port 8000 de la machine de dev à travers le câble USB. Dans ce cas, l'app peut utiliser `http://localhost:8000/api` comme si elle tournait sur la machine elle-même. À relancer après chaque redémarrage/rebranchement du téléphone.

### 2.7 Lancer l'app sur le téléphone

```bash
# Identifier l'ID exact du device si plusieurs sont connectés
flutter devices

# App citoyen
flutter run --flavor citoyen -t lib/main_citoyen.dart -d <device_id>

# App chauffeur
flutter run --flavor chauffeur -t lib/main_chauffeur.dart -d <device_id>
```

L'app s'installe et se lance directement sur le Samsung. Le hot reload (`r` dans le terminal) fonctionne normalement à travers le câble USB.

### 2.8 Tester le flux complet en local

1. S'assurer que `docker compose up` tourne (Phase 1) avec au minimum `db`, `redis`, `mosquitto`, `osrm`, `backend`, `mqtt_listener`.
2. Lancer l'app chauffeur sur le téléphone → se connecter (OTP Firebase, utiliser un numéro de test configuré dans la console Firebase) → démarrer une tournée.
3. Vérifier dans `docker compose logs -f mqtt_listener` que les positions sont bien reçues et traitées.
4. Lancer l'app citoyen (sur le même téléphone en changeant de flavor, ou sur un second appareil si disponible) → enregistrer un point (domicile) proche de la position simulée du chauffeur.
5. Vérifier la réception d'une notification FCM sur le téléphone (nécessite Google Play Services, présent par défaut sur un Samsung grand public).

---

## PHASE 3 — INFRASTRUCTURE VPS

### 3.1 Choisir et créer le VPS

Le projet embarque OSRM (données OSM chargées en mémoire) en plus de Django/PostGIS/Redis/Mosquitto — prévoir plus de RAM qu'un projet web classique.

**Recommandé : Hetzner Cloud** (https://hetzner.com/cloud)

| Plan | vCPU | RAM | SSD | Prix | Usage |
|---|---|---|---|---|---|
| CX32 | 4 | 8 GB | 80 GB | ~6.90€/mois | MVP quartier pilote |
| CX42 | 8 | 16 GB | 160 GB | ~13€/mois | Si extrait OSM plus large ou plusieurs villes |

**Procédure :**
1. Créer un compte → vérifier l'email.
2. Nouveau projet → "Add Server".
3. **Location** : proche de l'Europe de l'Ouest (meilleure latence Afrique de l'Ouest que les datacenters US).
4. **Image** : `Ubuntu 24.04`.
5. **Type** : CX32 minimum.
6. **SSH Key** : ajouter ta clé publique (voir 3.2).
7. **Name** : `autombalit-prod`.
8. "Create & Buy Now" → noter l'**IP publique**.

> Alternatives : DigitalOcean, Vultr, eVPS.net — Ubuntu 24.04 partout, le reste du guide est identique.

### 3.2 Générer et ajouter ta clé SSH

```bash
ls ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -C "autombalit-vps"
cat ~/.ssh/id_ed25519.pub
```

Copier le contenu affiché dans le champ SSH Key du fournisseur VPS.

### 3.3 Acheter un nom de domaine

**Recommandé : Namecheap** (https://namecheap.com)

Procédure identique quel que soit le registrar : chercher le nom, ajouter uniquement le domaine au panier, décocher SSL/hosting/email additionnels (Certbot fera le SSL gratuitement), payer.

### 3.4 Configurer les DNS

Chez le registrar → gestion DNS du domaine → supprimer les enregistrements existants, ajouter :

| Type | Host | Value | TTL |
|---|---|---|---|
| A Record | `@` | `TON_IP_VPS` | Automatic |
| A Record | `www` | `TON_IP_VPS` | Automatic |
| A Record | `api` | `TON_IP_VPS` | Automatic |

Vérifier la propagation (5-30 min) :

```bash
ping ton-domaine.com
```

> ⚠️ Ne pas lancer Certbot avant propagation DNS complète.

---

## PHASE 4 — SERVEUR VPS (STACK SYSTÈME)

### 4.1 Connexion et sécurisation de base

```bash
ssh root@TON_IP_VPS
passwd   # si mot de passe fourni par défaut, le changer immédiatement
apt update && apt upgrade -y
```

Si "Pending kernel upgrade" :

```bash
reboot
# attendre 30s puis
ssh root@TON_IP_VPS
```

### 4.2 Paquets de base

```bash
apt install -y nginx redis-server git unzip curl supervisor cron ufw \
  build-essential python3.12 python3.12-venv python3-pip
```

### 4.3 PostgreSQL + PostGIS

```bash
apt install -y postgresql postgresql-contrib postgis postgresql-16-postgis-3
systemctl enable postgresql
```

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE autombalit_prod;
CREATE USER autombalit_user WITH PASSWORD 'MOT_DE_PASSE_FORT_ICI';
GRANT ALL PRIVILEGES ON DATABASE autombalit_prod TO autombalit_user;
\c autombalit_prod
CREATE EXTENSION postgis;
\q
```

> ⚠️ Si le mot de passe contient des caractères spéciaux, le mettre entre guillemets dans `.env`.

### 4.4 Mosquitto (MQTT)

```bash
apt install -y mosquitto mosquitto-clients
systemctl enable mosquitto
```

Configurer l'authentification et l'ACL :

```bash
mosquitto_passwd -c /etc/mosquitto/passwd autombalit_backend
# entrer un mot de passe fort — c'est le compte utilisé par le backend pour s'abonner à tout

nano /etc/mosquitto/conf.d/autombalit.conf
```

```conf
allow_anonymous false
password_file /etc/mosquitto/passwd
acl_file /etc/mosquitto/acl

listener 1883 localhost
listener 8883
certfile /etc/letsencrypt/live/ton-domaine.com/fullchain.pem
keyfile /etc/letsencrypt/live/ton-domaine.com/privkey.pem
```

```bash
nano /etc/mosquitto/acl
```

```
# Le backend peut tout lire
user autombalit_backend
topic read camions/#

# Un compte MQTT par camion (pas par chauffeur, voir DECISIONS.md — Tâche 8) : l'assignation
# chauffeur/camion étant journalière, lier les credentials au camion évite une ACL dynamique.
# Une ligne par camion, ajoutée par le provisioning (voir ci-dessous) :
# user camion_<camion_id>
# topic write camions/<camion_id>/position
```

**Provisioning** : mêmes scripts qu'en local (`scripts/provision_mqtt_backend.sh`, `scripts/provision_mqtt_camion.sh`), à adapter en remplaçant `docker compose exec -T mosquitto mosquitto_passwd ...` par un appel direct à `mosquitto_passwd` (binaire installé nativement sur le VPS, pas de conteneur Docker pour Mosquitto en production) et en pointant vers `/etc/mosquitto/passwd`/`/etc/mosquitto/acl`. `systemctl restart mosquitto` remplace `docker compose restart mosquitto`.

> Le certificat SSL n'existe pas encore à cette étape (Phase 4.9) — laisser le bloc `listener 8883` en commentaire jusqu'à l'obtention du certificat, puis décommenter et redémarrer Mosquitto.

```bash
systemctl restart mosquitto
systemctl status mosquitto
```

### 4.5 Redis

```bash
systemctl enable redis-server
systemctl status redis-server
redis-cli ping   # doit répondre PONG
```

### 4.6 OSRM (routing)

Installer Docker si pas déjà présent :

```bash
curl -fsSL https://get.docker.com | sh
```

Préparer les données OSM (une fois, ou à chaque mise à jour d'extrait) :

```bash
mkdir -p /opt/osrm-data && cd /opt/osrm-data
curl -fLO https://download.geofabrik.de/africa/senegal-and-gambia-latest.osm.pbf

docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-extract -p /opt/car.lua /data/senegal-and-gambia-latest.osm.pbf
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-partition /data/senegal-and-gambia-latest.osrm
docker run -t -v "${PWD}:/data" osrm/osrm-backend osrm-customize /data/senegal-and-gambia-latest.osrm
```

Lancer OSRM en service permanent (conteneur qui redémarre automatiquement) :

```bash
docker run -d --name osrm-routed --restart unless-stopped \
  -p 127.0.0.1:5000:5000 \
  -v /opt/osrm-data:/data \
  osrm/osrm-backend osrm-routed --algorithm mld /data/senegal-and-gambia-latest.osrm
```

Vérifier :

```bash
curl "http://127.0.0.1:5000/route/v1/driving/-17.4467,14.6928;-17.44,14.70"
```

### 4.7 Firewall

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw allow 8883/tcp   # MQTT sécurisé (TLS), si les chauffeurs se connectent directement au VPS
ufw enable
ufw status
```

Ne **pas** exposer publiquement le port PostgreSQL (5432), Redis (6379), OSRM (5000) ou MQTT non chiffré (1883) — tous restent en écoute `127.0.0.1`/`localhost` uniquement.

---

## PHASE 5 — CODE EN PRODUCTION

### 5.1 Préparer le repo avant déploiement (en local)

Vérifier que `.gitignore` contient au minimum :

```
venv/
__pycache__/
*.pyc
.env
.env.*
!.env.example
firebase-service-account.json
staticfiles/
media/
osrm-data/
```

```bash
git add .
git commit -m "ready for production"
git push origin main
```

### 5.2 Cloner sur le VPS

```bash
mkdir -p /var/www && cd /var/www
git clone git@github.com:TON_COMPTE/autombalit-backend.git
cd autombalit-backend
```

Si repo privé et clone SSH : générer une clé SSH sur le VPS et l'ajouter comme *deploy key* GitHub (lecture seule) plutôt que la clé personnelle.

### 5.3 Environnement Python et dépendances

```bash
python3.12 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn daphne
```

### 5.4 Fichier `.env` production

```bash
cp .env.example .env
nano .env
```

```env
DEBUG=False
SECRET_KEY=GENERER_UNE_CLE_FORTE_ICI
ALLOWED_HOSTS=api.ton-domaine.com

DATABASE_URL=postgis://autombalit_user:MOT_DE_PASSE_FORT_ICI@localhost:5432/autombalit_prod

REDIS_URL=redis://localhost:6379/0

MQTT_BROKER_HOST=localhost
MQTT_BROKER_PORT=1883
MQTT_USERNAME=autombalit_backend
MQTT_PASSWORD=MOT_DE_PASSE_MQTT_ICI

OSRM_BASE_URL=http://127.0.0.1:5000

FIREBASE_CREDENTIALS_JSON=/var/www/autombalit-backend/firebase-service-account.json
FCM_PROJECT_ID=ton-projet-firebase
```

Générer une `SECRET_KEY` forte :

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

Transférer `firebase-service-account.json` sur le VPS de façon sécurisée (`scp`, jamais par un canal non chiffré) :

```bash
scp firebase-service-account.json root@TON_IP_VPS:/var/www/autombalit-backend/
```

### 5.5 Migrations et fichiers statiques

```bash
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

---

## PHASE 6 — SERVICES PRODUCTION

### 6.1 Supervisor — ASGI (Daphne), listener MQTT, worker Celery (notifications asynchrones)

```bash
nano /etc/supervisor/conf.d/autombalit-asgi.conf
```

```ini
[program:autombalit-asgi]
directory=/var/www/autombalit-backend
command=/var/www/autombalit-backend/venv/bin/daphne -b 127.0.0.1 -p 8001 autombalit_backend.asgi:application
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/www/autombalit-backend/logs/asgi.log
```

```bash
nano /etc/supervisor/conf.d/autombalit-mqtt.conf
```

```ini
[program:autombalit-mqtt]
directory=/var/www/autombalit-backend
command=/var/www/autombalit-backend/venv/bin/python manage.py run_mqtt_listener
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/www/autombalit-backend/logs/mqtt.log
```

Si le calcul ETA/envoi FCM est déporté en tâche asynchrone (recommandé pour ne pas bloquer le listener MQTT sous charge) :

```bash
nano /etc/supervisor/conf.d/autombalit-worker.conf
```

```ini
[program:autombalit-worker]
directory=/var/www/autombalit-backend
command=/var/www/autombalit-backend/venv/bin/celery -A autombalit_backend worker -l info
autostart=true
autorestart=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/autombalit-backend/logs/worker.log
```

```bash
mkdir -p /var/www/autombalit-backend/logs
chown -R www-data:www-data /var/www/autombalit-backend

supervisorctl reread
supervisorctl update
supervisorctl start all
supervisorctl status
```

Résultat attendu : tous les process en `RUNNING`.

### 6.2 Nginx (reverse proxy HTTP + WebSocket)

```bash
nano /etc/nginx/sites-available/autombalit
```

```nginx
server {
    listen 80;
    server_name api.ton-domaine.com;

    location /static/ {
        alias /var/www/autombalit-backend/staticfiles/;
    }

    location /ws/ {
        proxy_pass http://127.0.0.1:8001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/autombalit /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

### 6.3 SSL avec Certbot

> ⚠️ Uniquement après propagation DNS complète.

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d api.ton-domaine.com
certbot renew --dry-run   # doit afficher "All simulated renewals succeeded"
```

Une fois le certificat obtenu, revenir sur `/etc/mosquitto/conf.d/autombalit.conf` (Phase 4.4), décommenter le `listener 8883` avec les chemins de certificat corrects, puis :

```bash
systemctl restart mosquitto
```

### 6.4 Nettoyage périodique (rétention des positions)

Conformément à la règle métier de rétention (`SPEC.md` §3 — historique brut 24-48h max), planifier une tâche périodique :

```bash
crontab -e -u www-data
```

```
0 3 * * * cd /var/www/autombalit-backend && /var/www/autombalit-backend/venv/bin/python manage.py purge_old_positions >> /var/www/autombalit-backend/logs/purge.log 2>&1
```

---

## PHASE 7 — VÉRIFICATION

```bash
# Services système
systemctl status nginx postgresql redis-server mosquitto

# Supervisor
supervisorctl status   # tout doit être RUNNING

# OSRM
curl "http://127.0.0.1:5000/route/v1/driving/-17.4467,14.6928;-17.44,14.70"

# API publique
curl -I https://api.ton-domaine.com/api/   # doit répondre en HTTPS

# MQTT (depuis le VPS)
mosquitto_sub -h localhost -p 1883 -u autombalit_backend -P MOT_DE_PASSE_MQTT_ICI -t "camions/#"
```

Logs en temps réel :

```bash
tail -f /var/www/autombalit-backend/logs/asgi.log
tail -f /var/www/autombalit-backend/logs/mqtt.log
tail -f /var/www/autombalit-backend/logs/worker.log
tail -f /var/log/nginx/error.log
```

---

## PHASE 8 — MAINTENANCE

### Script de déploiement (`deploy.sh`)

```bash
nano /var/www/autombalit-backend/deploy.sh
```

```bash
#!/bin/bash
set -e

echo "🚀 Déploiement en cours..."
cd /var/www/autombalit-backend
source venv/bin/activate

echo "📥 Pull du code..."
git pull origin main

echo "📦 Dépendances..."
pip install -r requirements.txt

echo "🗄️ Migrations..."
python manage.py migrate

echo "🎨 Fichiers statiques..."
python manage.py collectstatic --noinput

echo "🔄 Redémarrage services..."
supervisorctl restart all

echo "✅ Déploiement terminé !"
```

```bash
chmod +x /var/www/autombalit-backend/deploy.sh
```

Utilisation à chaque mise à jour :

```bash
bash /var/www/autombalit-backend/deploy.sh
```

### Commandes utiles au quotidien

```bash
supervisorctl restart autombalit-mqtt      # redémarrer uniquement le listener MQTT
supervisorctl restart autombalit-asgi      # redémarrer uniquement le serveur ASGI
certbot renew                                # renouveler SSL manuellement si besoin
df -h                                        # espace disque (attention aux données OSM + logs)
free -h                                      # mémoire RAM (OSRM est gourmand)
htop                                         # processus actifs
docker logs osrm-routed                      # logs du conteneur OSRM
```

### Points d'attention fréquents

| Problème | Cause probable | Solution |
|---|---|---|
| WebSocket ne se connecte pas | Bloc `/ws/` manquant côté Nginx ou Daphne down | Vérifier `supervisorctl status autombalit-asgi`, vérifier le bloc Nginx |
| Positions non reçues côté backend | `autombalit-mqtt` arrêté ou mauvais identifiants MQTT | `supervisorctl status`, vérifier `.env` vs `/etc/mosquitto/passwd` |
| ETA absurde ou vide | OSRM down ou extrait OSM absent | `docker ps`, `curl` direct sur le port 5000 |
| Notification jamais envoyée | Token FCM absent/expiré ou credentials Firebase invalides | Vérifier `firebase-service-account.json`, logs du worker |
| `nginx -t` échoue | Certificat SSL pas encore généré | Configurer en HTTP d'abord, lancer Certbot ensuite |
| RAM saturée | OSRM + Postgres + Redis sur un VPS trop petit | Passer au plan CX42, ou réduire l'extrait OSM à la région pilote |

---

## PHASE 9 — PUBLICATION MOBILE (ANDROID)

### 9.1 Générer une clé de signature

```bash
keytool -genkey -v -keystore ~/autombalit-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias autombalit
```

Conserver ce fichier et son mot de passe en lieu sûr — sa perte empêche toute mise à jour future de l'app publiée.

### 9.2 Configurer la signature dans le projet Flutter

`android/key.properties` (non commité) :

```
storePassword=MOT_DE_PASSE
keyPassword=MOT_DE_PASSE
keyAlias=autombalit
storeFile=/chemin/absolu/vers/autombalit-release.jks
```

### 9.3 Build de production

```bash
# App citoyen
flutter build appbundle --flavor citoyen -t lib/main_citoyen.dart --release

# App chauffeur
flutter build appbundle --flavor chauffeur -t lib/main_chauffeur.dart --release
```

Fichiers générés sous `build/app/outputs/bundle/<flavor>Release/app-<flavor>-release.aab`.

### 9.4 Publication sur Google Play Console

1. Créer un compte développeur Google Play (frais unique ~25 $).
2. Créer une nouvelle application → uploader le `.aab`.
3. Remplir fiche store (description, captures d'écran, politique de confidentialité — lien vers la page rédigée en Phase 0 du projet, voir `TODO.md`).
4. Pour l'app chauffeur : envisager une **distribution fermée/interne** (pas de Play Store public) via un canal de test restreint, puisque son usage est réservé aux chauffeurs validés par une société de collecte.
5. Soumettre pour revue.

---

## PHASE 10 — UTILISATION FINALE (PARCOURS UTILISATEUR)

### Côté admin société / mairie (web admin)

1. Se connecter à `https://admin.ton-domaine.com` avec le compte créé via `createsuperuser` ou un compte admin société dédié.
2. Uploader les fichiers GeoJSON des tournées et zones fournis par l'équipe SIG.
3. Configurer le calendrier de collecte par zone (jours de passage).
4. Valider les comptes chauffeurs en attente (`statut_validation: en_attente → valide`).
5. Suivre le tableau de bord des signalements remontés par les citoyens.

### Côté chauffeur (app mobile)

1. Installer l'app chauffeur (distribution interne/APK direct pour le pilote, Play Store en fermé ensuite).
2. S'inscrire avec son numéro de téléphone (OTP).
3. Attendre la validation par l'admin société (statut visible dans l'app).
4. Une fois validé, consulter sa tournée assignée du jour.
5. Appuyer sur "Démarrer la tournée" au départ — l'app publie alors sa position à fréquence adaptative.
6. Appuyer sur "Arrêter la tournée" à la fin — le tracking GPS cesse immédiatement.

### Côté citoyen (app mobile)

1. Installer l'app citoyen (Play Store public).
2. S'inscrire avec son numéro de téléphone (OTP).
3. Enregistrer son domicile (recherche d'adresse ou pointage sur la carte).
4. Consulter l'écran principal : calendrier de collecte de sa zone, dernier passage connu.
5. Recevoir des notifications automatiques à 30/20/10/5 minutes de l'arrivée estimée du camion les jours de collecte.
6. Ouvrir la carte live (optionnel) pour voir la position du camion en temps réel pendant qu'il approche.
7. Signaler un problème (camion non passé, notification manquante, position incohérente) via le formulaire dédié si besoin.

---

## RÉCAPITULATIF — CE QUI TOURNE EN PRODUCTION

| Service | Gestionnaire | Rôle |
|---|---|---|
| Nginx | systemd | HTTP/HTTPS + proxy WebSocket |
| PostgreSQL + PostGIS | systemd | Base de données géospatiale |
| Redis | systemd | Cache + couche Channels (WebSocket) + broker Celery |
| Mosquitto | systemd | Broker MQTT (chauffeur → backend) |
| OSRM | Docker (`--restart unless-stopped`) | Calcul de distance/temps routier |
| Daphne (ASGI) | Supervisor | Serveur applicatif Django (HTTP + WebSocket) |
| `run_mqtt_listener` | Supervisor | Consommation des positions MQTT |
| Celery worker | Supervisor | Calcul ETA asynchrone + envoi FCM |
| Cron (`purge_old_positions`) | crontab www-data | Rétention des positions (24-48h) |
| Certbot | timer systemd | Renouvellement SSL automatique |

## COÛT ESTIMÉ

| Poste | Coût |
|---|---|
| VPS Hetzner CX32 | ~6.90€/mois |
| Domaine .com | ~8€/an |
| SSL Certbot | Gratuit |
| Firebase Auth + FCM | Gratuit (usage MVP) |
| OSRM (self-hosted) | Gratuit (inclus dans le VPS) |
| Google Play Console | 25 $ (frais unique) |
| **Total minimal** | **~7-8€/mois + frais ponctuels** |
