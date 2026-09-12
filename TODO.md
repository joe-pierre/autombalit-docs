# TODO

## Phase 0 — Cadrage

- [x] Faire valider "AutoMbalit" par un locuteur wolof natif (naturel de l'expression, connotations éventuelles) — confirmé, le nom sonne bien
- [ ] Vérifier la disponibilité du nom de domaine, des comptes réseaux sociaux et des identifiants Google Play/App Store pour "AutoMbalit"
- [ ] Choisir un quartier pilote
- [ ] Identifier une société de collecte partenaire (ou un camion volontaire)
- [ ] Vérifier la couverture/qualité OSM sur le quartier pilote
- [ ] Rédiger une politique de confidentialité minimale (page simple)
- [ ] Vérifier a minima si une déclaration CDP Sénégal est nécessaire (consultation légère)

## Phase 1 — Fondations backend

- [x] Écrire `docker-compose.yml` (db PostGIS, redis, mosquitto, osrm, backend, mqtt_listener) et `Dockerfile` backend
- [x] Setup projet Django + PostgreSQL/PostGIS + GeoDjango
- [x] Implémenter les modèles de données (Societe, Chauffeur, Camion, Zone, Tournee, TourneeZone, CalendrierCollecte, AssignationJournaliere, PositionCamion, Utilisateur, PointEnregistre, Signalement)
- [x] Migrations + vérification PostGIS fonctionnelle (via `docker compose exec backend`)
- [x] Endpoints DRF de base : CRUD tournées/zones, auth JWT
- [x] Endpoints DRF de base : upload GeoJSON
- [x] Validation GeoJSON à l'upload (SRID 4326, type de géométrie, properties requises)
- [x] Préparer les données OSRM localement (extrait région pilote) et vérifier le service `osrm` dans `docker compose`
- [x] Valider que toute la stack (web + base) est exploitable en local via `docker compose up` avant toute mise en production

## Phase 2 — Cœur temps réel

- [x] Setup broker MQTT (Mosquitto) + authentification par camion + ACL par topic
- [x] Handler Django : consommation des positions MQTT → écriture `PositionCamion`
- [x] Intégration OSRM pour le calcul de distance/temps
- [x] Facteur de correction historique par zone (valeur par défaut le temps d'accumuler l'historique)
- [x] Logique de seuils (30/20/10/5 min) avec règle anti-spam (une notif/seuil/jour/utilisateur)
- [x] Gestion des positions en rafale (ne traiter que la plus récente pour l'ETA/notification)
- [x] Intégration Firebase Auth (OTP téléphone) côté backend
- [x] Intégration FCM (envoi de notifications)

## Phase 3 — Apps mobiles (Flutter)

- [x] Activer le mode développeur + débogage USB sur le Samsung, vérifier `adb devices`
- [x] Setup projet Flutter (codebase unique, flavors chauffeur/citoyen)
- [ ] App Chauffeur : auth OTP, écran tournée du jour, start/stop tournée
- [ ] App Chauffeur : capture GPS + publication MQTT (fréquence adaptative)
- [ ] App Chauffeur : queue locale hors-ligne (stockage local + retry MQTT QoS 1/2)
- [ ] App Citoyen : auth OTP, enregistrement du domicile (flutter_map)
- [ ] App Citoyen : réception et affichage des notifications FCM
- [ ] App Citoyen : écran statut du jour (calendrier + dernier passage connu, cache local)
- [ ] App Citoyen : carte live optionnelle (WebSocket, à la demande)
- [ ] App Citoyen : formulaire de signalement

## Phase 4 — Web admin

- [ ] Trancher le choix frontend web admin (Django templates vs templates + htmx)
- [ ] Interface d'upload GeoJSON + visualisation carte (Leaflet)
- [ ] Gestion des calendriers de collecte par zone
- [ ] Validation des comptes chauffeurs
- [ ] Tableau de bord des signalements par zone/société

## Phase 5 — Test pilote réel

- [ ] Déploiement sur le quartier pilote avec un camion volontaire
- [ ] Mesure de la précision réelle de l'ETA
- [ ] Ajustement du facteur de correction par zone à partir des données terrain
- [ ] Collecte et analyse des premiers signalements
- [ ] Décision go/no-go pour extension à d'autres quartiers/sociétés

## Backlog / non planifié

- [ ] Support iOS (après validation MVP Android)
- [ ] Optimisation de tournées (VRP) — V2
- [ ] Consolidation multi-société pour l'app citoyen (une seule app, plusieurs sociétés couvrant des zones différentes)
- [ ] Modèle économique / contractualisation (mairie, société, subvention)
