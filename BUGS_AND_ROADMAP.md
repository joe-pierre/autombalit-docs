# BUGS CORRIGÉS

Voir `DECISIONS.md` (entrées `[RÉSOLU]`) pour l'historique des bugs corrigés au fil des tâches.

# BUGS OUVERTS

## `mqtt_listener` (backend) ne se reconnecte jamais automatiquement à Mosquitto après un redémarrage du broker

**Découvert :** Tâche 16 (`autombalit-mobile`), lors du test manuel de la queue hors-ligne — voir `DECISIONS.md` pour le détail complet.
**Symptôme :** après un arrêt/redémarrage du broker Mosquitto (`docker compose stop/start mosquitto`), le service `mqtt_listener` reste connecté à son état "déconnecté" (`Déconnecté du broker MQTT : Unspecified error` en boucle dans ses logs) sans jamais retenter la connexion — un `docker compose restart mqtt_listener` manuel est nécessaire pour qu'il recommence à recevoir des positions.
**Impact :** toute coupure du broker (redémarrage, crash, maintenance) fait perdre silencieusement toutes les positions publiées par les camions tant que `mqtt_listener` n'est pas relancé manuellement — y compris celles republiées avec succès (ack reçu) par la queue hors-ligne côté app chauffeur (Tâche 16), puisque MQTT ne rejoue pas les messages à un abonné absent au moment de la publication.
**Non corrigé** — hors périmètre de la Tâche 16 (mobile). À traiter dans une tâche dédiée côté `autombalit-backend` (probablement `client.on_disconnect` + logique de reconnexion dans `tracking/clients/mqtt_client.py`, Tâche 9).

# ROADMAP (idées / améliorations futures)

## V2 — après validation du MVP pilote

- Exploiter les `Signalement` pour affiner automatiquement le facteur de correction ETA par zone (au lieu d'un ajustement manuel).
- Notification spéciale "camion arrêté à proximité" si le camion reste immobile longtemps très près d'un utilisateur enregistré, pour réduire la frustration d'attente.
- Tableau de bord analytics plus riche côté web admin (heures moyennes de passage, taux de fiabilité par zone, statistiques de signalements).
- Optimisation de tournées (VRP — Vehicle Routing Problem) une fois l'historique de positions suffisant, potentiellement via GraphHopper.
- Support iOS (une fois le MVP Android validé sur le terrain).

## Idées ouvertes / à évaluer plus tard

- Consolidation multi-société dans une seule app citoyen si plusieurs sociétés de collecte couvrent des zones différentes d'une même ville (éviter la fragmentation évoquée au point gouvernance).
- Contribution/correction collaborative des données OpenStreetMap sur les zones cibles mal cartographiées.
- Self-host de tuiles OSM (TileServer GL) si le volume d'usage rend la dépendance aux tuiles publiques problématique.
- Pondération de la fiabilité d'un signalement si un utilisateur en soumet un volume anormalement élevé (anti-abus avancé).
- Étude de modèle économique (abonnement B2B société de collecte, contrat mairie, subvention/ONG) une fois la preuve de concept validée.
- Clarification contractuelle de la gouvernance des données si plusieurs sociétés/mairies utilisent la plateforme (propriété, export, réutilisation).

## Points légaux/confidentialité à traiter avant un lancement public (non bloquant pour le MVP pilote)

- Vérification formelle auprès de la CDP (Commission de Protection des Données Personnelles, Sénégal) sur la nécessité d'une déclaration pour le traitement de géolocalisation.
- Rédaction d'une politique de confidentialité complète (au-delà de la version minimale de la Phase 0).
