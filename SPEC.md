# SPEC.md — Spécifications fonctionnelles et techniques

## 1. Vue d'ensemble

**AutoMbalit** est une application de géolocalisation des camions de collecte d'ordures au Sénégal. Elle remplace le klaxon des camions (nuisance sonore, absence d'information fiable) par un système de notifications basé sur le temps estimé d'arrivée (ETA).

**Problème résolu** : savoir si le camion poubelle va bientôt arriver, est déjà passé, ou ne passera pas aujourd'hui.

**Composants du système** :
- **App Flutter Citoyen** : enregistre un point fixe (domicile), reçoit des notifications par seuil ETA (30/20/10/5 min), carte live optionnelle.
- **App Flutter Chauffeur** : démarre/arrête sa tournée, publie sa position GPS pendant qu'elle est active.
- **Web Admin (SIG)** : upload des tracés GeoJSON pré-convertis par l'équipe SIG, gestion des tournées/zones/calendriers, validation des comptes chauffeurs.
- **Backend Django** : cœur logique — calcul d'ETA, gestion des seuils/notifications, stockage géospatial, API REST et temps réel.

**Contraintes structurantes** :
- Budget de développement nul : uniquement des outils gratuits/self-hostables.
- Contexte sénégalais : connectivité mobile instable, couverture OSM variable selon les quartiers, marché mobile dominé par Android.
- Le métier de collecte implique des arrêts fréquents (tous les 30-50 m) : la vitesse instantanée n'est pas représentative, l'ETA doit intégrer un facteur de correction empirique.
- Le tracé GeoJSON d'une tournée est une **référence indicative**, pas un chemin strict (le chauffeur peut varier légèrement selon les circonstances).

## 2. Stack technique

| Composant | Choix | Statut |
|---|---|---|
| Backend | Django 5.x + Django REST Framework + GeoDjango | Validé |
| Base de données | PostgreSQL + PostGIS | Validé |
| Routing | OSRM self-hosted (données OSM via Geofabrik Sénégal) | Validé |
| Communication chauffeur → backend | MQTT (Mosquitto), QoS 1/2 | Validé |
| Temps réel citoyen (carte live) | Django Channels + Redis, WebSocket à la demande | Validé |
| Notifications | Firebase Cloud Messaging (FCM) | Validé |
| Authentification | Firebase Auth (OTP téléphone) | Validé |
| Jetons API | JWT via `djangorestframework-simplejwt` | Validé |
| Mobile | Flutter (codebase unique, chauffeur + citoyen, Android prioritaire) | Validé |
| Cartographie mobile | `flutter_map` + tuiles OpenStreetMap | Validé |
| Import géographique | GeoJSON pré-converti par l'équipe SIG, upload via web admin | Validé |
| Web admin | Django (templates ou DRF + frontend léger — à trancher en Phase 4) | Ouvert |

Aucune déviation de cette stack sans décision explicite consignée dans `DECISIONS.md`.

## 3. Modèle de données

### Entités principales

- **Societe** : société de collecte / mairie. `id, nom, contact`.
- **Chauffeur** : `id, nom, telephone, statut_validation (en_attente/valide/rejete), societe_id FK, firebase_uid`.
- **Camion** : `id, immatriculation, societe_id FK`.
- **Zone** : quartier/secteur. `id, nom, polygone (PolygonField)`.
- **Tournee** : `id, nom, trajet_reference (LineStringField, indicatif), societe_id FK`, liée à `Zone` en many-to-many via `TourneeZone`.
- **TourneeZone** : table de liaison `tournee_id, zone_id, ordre_passage`.
- **CalendrierCollecte** : `id, tournee_id FK, jour_semaine, heure_debut_estimee`.
- **AssignationJournaliere** : lien flexible camion ↔ chauffeur ↔ tournée pour une date donnée. `id, camion_id FK, chauffeur_id FK, tournee_id FK, date, statut`. Contrainte unique `(camion, date)`.
- **PositionCamion** : `id, camion_id FK, assignation_id FK (nullable), point (PointField), horodatage`. Index sur `(camion, -horodatage)`.
- **Utilisateur** (citoyen) : `id, telephone, firebase_uid, fcm_token`.
- **PointEnregistre** : `id, utilisateur_id FK, nom (ex: "Domicile"), point (PointField), zone_id FK (déduite automatiquement)`.
- **Signalement** : `id, utilisateur_id FK, camion_id FK (nullable), zone_id FK (nullable), type (pas_notifie/pas_passe/position_incoherente/autre), commentaire, horodatage`.

Schéma entité-relation et modèles Django complets déjà produits et validés — voir historique de conception. `django.contrib.gis` doit être activé, PostGIS doit être l'extension active sur la base.

### Politique de rétention

- Historique brut de `PositionCamion` : conservé 24-48h maximum.
- Au-delà : agrégats statistiques anonymisés (heure moyenne de passage par zone/jour), pas de position individuelle brute conservée indéfiniment.

## 4. Règles métier critiques

1. **Tracking chauffeur limité à la tournée active** : le GPS n'est capturé et publié que lorsque le chauffeur a explicitement démarré sa tournée (bouton start). Jamais de tracking en tâche de fond hors service.
2. **Aucun partage de position exacte du citoyen** : le backend seul calcule l'ETA à partir du point enregistré ; ni le chauffeur ni un autre utilisateur n'y a accès.
3. **Validation manuelle des chauffeurs** : un compte chauffeur est créé avec `statut_validation = en_attente` et ne peut publier aucune position tant qu'un admin société ne l'a pas validé (`valide`). Le backend doit rejeter toute publication MQTT d'un chauffeur non validé.
4. **ETA = distance routière (OSRM) + facteur de correction historique par zone**, pas de map-matching strict sur le tracé GeoJSON (le trajet réel varie). Valeur par défaut (~10 km/h zone résidentielle) tant que l'historique est insuffisant (< 5-10 passages enregistrés).
5. **Anti-spam des notifications** : un seul envoi par seuil (30/20/10/5 min) par couple `(utilisateur, camion_du_jour)` et par jour. Reset quotidien de la liste des seuils notifiés. Pas de re-notification si l'ETA remonte au-dessus d'un seuil déjà notifié puis redescend.
6. **Positions reçues en rafale après coupure réseau** : seule la position au timestamp le plus récent du lot déclenche un recalcul d'ETA/notification. Les positions plus anciennes sont stockées pour historique/statistiques mais ignorées pour le déclenchement temps réel.
7. **Un camion = une assignation par date** (`AssignationJournaliere` unique sur `camion + date`) : la relation camion/tournée/chauffeur est journalière, pas fixe.
8. **Une tournée peut couvrir plusieurs zones** dans un ordre donné (`TourneeZone.ordre_passage`), et une zone peut être couverte par plusieurs tournées.
9. **Calendrier de collecte indépendant du GPS** : la réponse à « passera-t-il aujourd'hui ? » s'appuie d'abord sur `CalendrierCollecte`, pas uniquement sur la présence d'une position GPS récente.

## 5. Architecture code

### Backend Django (apps proposées)

```
autombalit_backend/
├── core/            # Societe, Chauffeur, Camion, Zone, Tournee, TourneeZone, CalendrierCollecte
├── tracking/         # AssignationJournaliere, PositionCamion, consommateur MQTT, calcul ETA
├── citizens/         # Utilisateur, PointEnregistre, Signalement
├── notifications/    # intégration FCM, logique de seuils
├── realtime/          # Django Channels — consumers WebSocket, groupes
├── geo_import/       # upload/validation GeoJSON, endpoints web admin
└── api/               # serializers DRF, routers, permissions par rôle
```

Chaque app suit une architecture en couches — modèles (structure des données), services (logique métier), selectors (lectures complexes), clients (intégrations externes encapsulées : OSRM, FCM, MQTT, Firebase), vues/serializers (fins, sans logique métier). Détail des règles et de la limite de taille par fichier dans `CONVENTIONS.md` §Architecture et paradigmes — objectif : aucun god file, aucun god model, logique métier testable indépendamment du framework web.

### Mobile Flutter (codebase unique, flavors)

```
lib/
├── common/           # auth Firebase, client API, modèles partagés, thème
├── driver/           # écran start/stop tournée, capture GPS, queue MQTT locale hors-ligne
├── citizen/          # enregistrement domicile, réception FCM, carte live optionnelle
└── shared_widgets/   # flutter_map, composants UI communs
```

Deux points d'entrée/flavors (`chauffeur`, `citoyen`) partageant `common/` et `shared_widgets/`.

## 6. Événements WebSocket (Django Channels)

- **Groupe** : un groupe par zone ou par camion actif, ex. `camion_<camion_id>` ou `zone_<zone_id>`.
- **Connexion** : uniquement à l'ouverture de l'écran carte live côté app citoyen ; fermeture automatique à la sortie de l'écran.
- **Événement `position.update`** : `{ "camion_id": ..., "lat": ..., "lng": ..., "horodatage": ... }`, diffusé au groupe concerné à chaque position MQTT reçue et traitée par le backend.
- **Événement `tournee.status`** (optionnel V2) : changement de statut d'une assignation (`en_cours`, `terminee`).
- Le canal MQTT chauffeur → backend reste interne ; le citoyen ne se connecte jamais directement au broker MQTT.

## 7. Endpoints API (DRF)

### Auth
- `POST /api/auth/token/` — échange token Firebase → JWT applicatif.
- `POST /api/auth/token/refresh/`

### Citoyen
- `GET/POST /api/points-enregistres/` — CRUD des points enregistrés de l'utilisateur connecté.
- `DELETE /api/points-enregistres/{id}/`
- `GET /api/zones/{id}/calendrier/` — calendrier de collecte de la zone.
- `POST /api/signalements/` — créer un signalement.
- `DELETE /api/me/` — suppression de compte/données (RGPD/CDP).

### Chauffeur
- `POST /api/tournees/du-jour/` — récupérer l'assignation du jour du chauffeur connecté.
- `POST /api/tournees/{id}/start/` — démarrer une tournée (ouvre la fenêtre de tracking).
- `POST /api/tournees/{id}/stop/` — arrêter une tournée.

### Web admin (société/mairie)
- `POST /api/admin/geojson/upload/` — upload GeoJSON validé (tournée ou zone).
- `CRUD /api/admin/tournees/`, `/api/admin/zones/`, `/api/admin/calendriers/`
- `POST /api/admin/chauffeurs/{id}/valider/` — validation manuelle d'un chauffeur.
- `GET /api/admin/signalements/` — tableau de bord des signalements par zone/société.

### Interne (non exposé publiquement)
- Handler MQTT → écriture `PositionCamion`, appel OSRM, vérification des seuils, déclenchement FCM.

## 8. Extensibilité

- Multi-société / multi-ville dès la conception (`Societe` comme entité racine).
- iOS repoussé après validation Android du MVP — Flutter permet l'extension sans réécriture.
- Optimisation de tournées (VRP) envisageable en V2 en s'appuyant sur GraphHopper ou une lib d'optimisation, une fois l'historique de positions suffisant.
- Signalements comme socle pour affiner automatiquement le facteur de correction ETA par zone.

## 9. Sécurité et validations

- **JWT** : durée de vie courte + refresh token, `djangorestframework-simplejwt`.
- **MQTT** : authentification obligatoire (utilisateur/mot de passe ou certificats) + ACL Mosquitto restreignant chaque chauffeur à son propre topic `camions/{camion_id}/position`.
- **Validation GeoJSON à l'upload** : rejeter tout fichier non EPSG:4326, vérifier le type de géométrie attendu (`LineString`/`MultiLineString` pour une tournée, `Polygon` pour une zone), exiger une structure de `properties` définie (nom, zone, jours de collecte).
- **Anti-abus signalements** : un signalement par utilisateur, par camion, par type, par jour.
- **Aucune position brute citoyenne transmise à un tiers** — uniquement l'ETA calculé.
- **Consentement et information** : politique de confidentialité minimale exposée dans les deux apps (données collectées, finalité, durée de conservation, suppression de compte).

## 10. Identité visuelle

Direction retenue : **coloré et accessible**, pensé pour un public large (tous âges, tous niveaux d'aisance avec un smartphone) plutôt qu'un style "startup tech" sobre.

**Palette**
- App citoyen : vert dominant (`#639922`/`#3B6D11` sur fond `#EAF3DE`) — évoque la propreté/l'écologie, cohérent avec l'objet de l'app.
- App chauffeur : ambre/orange dominant (`#BA7517`/`#854F0B` sur fond `#FAEEDA`) — distingue visuellement les deux apps si installées sur le même appareil (utile en test), rappelle la visibilité/le métier du camion.
- Rouge (`#E24B4A`/`#A32D2D`) réservé aux alertes et à l'ETA imminent (<5 min) — jamais utilisé comme couleur de fond par défaut.
- Web admin : peut reprendre une base plus neutre (gris/bleu institutionnel), à confirmer en Phase 4.

**Code couleur de l'ETA (citoyen)** : vert (loin) → ambre (proche) → rouge (imminent), affiché en dégradé progressif à mesure que le camion approche. Toujours doublé d'une icône et d'un texte explicite (ex. "12 min") — jamais la couleur seule, pour rester lisible en cas de daltonisme ou de faible littératie.

**Typographie**
- Taille de base généreuse (16-18sp texte courant, 22-26sp titres), deux graisses seulement (regular/medium) — pas de texte fin ou gris pâle sur fond clair.
- Respecter l'échelle de police système de l'utilisateur (pas de taille verrouillée en pixels).

**Iconographie**
- Icônes pleines/arrondies plutôt que traits fins.
- Toute action importante (ex. "Démarrer la tournée", "Signaler un problème") est accompagnée d'un libellé texte — jamais d'icône seule pour une action critique.

**Zones tactiles et layout**
- Cibles tactiles 48×48dp minimum.
- Cartes larges plutôt que listes denses — préférer la clarté à la densité d'information.

**Accessibilité**
- Contraste minimum WCAG AA (4.5:1 pour le texte courant).
- Labels vocaux (`Semantics` Flutter) sur tous les éléments interactifs pour les lecteurs d'écran (TalkBack).
- Ne jamais faire reposer une information critique sur la couleur seule.

Web admin (Phase 4) et déclinaison iconographique complète (logo, app icons) restent à définir — cette section couvre la direction de design système, pas les assets finaux.

Contrainte technique inchangée : pas de dépendance à un SDK cartographique payant (pas de Google Maps SDK), rendu via `flutter_map`/OSM (voir `SPEC.md` §2, `DECISIONS.md`).

**Glassmorphism (usage mesuré)**
- Autorisé uniquement sur des éléments secondaires/décoratifs : panneau flottant d'information sur la carte live (position du camion par-dessus le fond de carte), barre du haut, barre de navigation basse.
- Interdit sur tout élément critique pour l'information ou l'action : carte ETA, calendrier de collecte, boutons d'action ("Démarrer la tournée", "Signaler un problème") — ces éléments restent opaques et à contraste maximal, conformément à la contrainte d'accessibilité ci-dessus.
- Paramètres indicatifs : flou léger (`sigma` ~8-10px), fond à 55-75% d'opacité, bordure fine (0.5-1px) semi-transparente pour définir le bord plutôt qu'une ombre portée.
- Implémentation Flutter : `BackdropFilter(filter: ImageFilter.blur(...))` dans un `ClipRRect` à coins arrondis, posé sur un `Container` semi-transparent.
- Vérifier systématiquement le contraste du texte posé sur un panneau vitré (fond variable selon ce qu'il y a dessous) — prévoir un fond de secours plus opaque si le contraste tombe sous le seuil WCAG AA à un endroit donné de la carte.

## 11. Écrans

**App Citoyen**
- Onboarding / auth OTP
- Enregistrement du domicile (carte + recherche)
- Écran principal : statut du jour (calendrier + dernier passage connu)
- Carte live optionnelle (camion en temps réel, uniquement à l'ouverture)
- Notifications reçues (historique local)
- Signalement d'un problème
- Paramètres / suppression de compte

**App Chauffeur**
- Auth OTP + statut de validation (en attente / validé)
- Écran tournée du jour (assignation reçue)
- Bouton start/stop tournée
- Indicateur d'état (GPS actif, file d'attente hors-ligne en cours d'envoi)

**Web Admin**
- Connexion admin société
- Upload GeoJSON (tournées, zones)
- Gestion des calendriers de collecte
- Validation des comptes chauffeurs
- Tableau de bord des signalements

## 12. Tâches

Voir `TODO.md` pour le découpage détaillé par phase, et `TASK_PROMPTS.md` pour les prompts destinés à Claude Code.

## 13. Race conditions

- **Positions en rafale après reconnexion réseau** : ne traiter que le timestamp le plus récent pour le calcul ETA/notification (règle métier §4.6) ; verrou ou vérification côté handler MQTT pour éviter un recalcul par position historique.
- **Double assignation d'un camion le même jour** : contrainte unique `(camion, date)` sur `AssignationJournaliere` au niveau base de données, pas seulement applicatif.
- **Notification envoyée deux fois pour le même seuil** en cas de traitement concurrent de deux positions proches dans le temps : la vérification « seuil déjà notifié » et son écriture doivent être atomiques (transaction/lock au niveau utilisateur+camion+jour) pour éviter une race entre deux workers.
- **Changement de statut de tournée pendant l'envoi d'une position** (ex: stop tournée au moment où une position est en transit) : le backend doit vérifier que l'assignation est toujours `en_cours` avant de traiter une position, sinon la journaliser sans déclencher de notification.
- **Suppression de compte utilisateur pendant un calcul ETA en cours** : gérer proprement la suppression en cascade / l'absence de token FCM sans faire échouer le job de notification global.
