# BUGS CORRIGÉS

Voir `DECISIONS.md` (entrées `[RÉSOLU]`) pour l'historique des bugs corrigés au fil des tâches.

# BUGS OUVERTS

Aucun confirmé pour l'instant. Point de vigilance (pas un bug ouvert actif) : la cause exacte de la perte de positions du test terrain de la Tâche 16 n'a pas pu être établie avec certitude malgré investigation (voir `DECISIONS.md`, entrée « Reconnexion automatique du `mqtt_listener`... »). Un durcissement a été appliqué (backoff de reconnexion borné, logs de reconnexion visibles) mais sans preuve qu'il cible le mécanisme exact d'origine — à surveiller lors du pilote (voir hypothèse `keepalive` MQTT ci-dessous).

# ROADMAP (idées / améliorations futures)

## V2 — après validation du MVP pilote

- Exploiter les `Signalement` pour affiner automatiquement le facteur de correction ETA par zone (au lieu d'un ajustement manuel).
- Notification spéciale "camion arrêté à proximité" si le camion reste immobile longtemps très près d'un utilisateur enregistré, pour réduire la frustration d'attente.
- Tableau de bord analytics plus riche côté web admin (heures moyennes de passage, taux de fiabilité par zone, statistiques de signalements).
- Optimisation de tournées (VRP — Vehicle Routing Problem) une fois l'historique de positions suffisant, potentiellement via GraphHopper.
- Support iOS (une fois le MVP Android validé sur le terrain).

## Idées ouvertes / à évaluer plus tard

- Accusé applicatif de bout en bout pour la publication de position (au-delà du simple PUBACK MQTT), si des pertes de positions sont à nouveau constatées en pilote malgré le fix de reconnexion du `mqtt_listener` — voir `DECISIONS.md` (« Limite structurelle : un ack MQTT ne garantit qu'une livraison au broker, jamais une persistance en base »).
- Réduire/configurer explicitement le `keepalive` MQTT de `mqtt_listener` (défaut paho actuel : 60s, jamais fixé explicitement dans `tracking/clients/mqtt_client.py`) si l'hypothèse d'une détection tardive de déconnexion (jusqu'à ~1,5× keepalive sans FIN/RST TCP immédiat) est un jour confirmée comme cause de perte de positions — non vérifiée à ce stade, voir `DECISIONS.md`.
- Consolidation multi-société dans une seule app citoyen si plusieurs sociétés de collecte couvrent des zones différentes d'une même ville (éviter la fragmentation évoquée au point gouvernance).
- Contribution/correction collaborative des données OpenStreetMap sur les zones cibles mal cartographiées.
- Self-host de tuiles OSM (TileServer GL) si le volume d'usage rend la dépendance aux tuiles publiques problématique.
- Pondération de la fiabilité d'un signalement si un utilisateur en soumet un volume anormalement élevé (anti-abus avancé).
- Étude de modèle économique (abonnement B2B société de collecte, contrat mairie, subvention/ONG) une fois la preuve de concept validée.
- Clarification contractuelle de la gouvernance des données si plusieurs sociétés/mairies utilisent la plateforme (propriété, export, réutilisation).
- **Détection automatique du retour de connectivité côté app citoyen** (écran statut du jour, Tâche 19) : vérifié qu'aucune logique de ce type n'existe (pas de `connectivity_plus` ni d'écoute équivalente, `DECISIONS.md`) — `StatutDuJourProvider.charger()` n'est déclenché qu'au démarrage de l'app ou par un tir-pour-rafraîchir manuel (`RefreshIndicator`). En usage réel (coupures réseau fréquentes au Sénégal, SPEC.md §1), un citoyen qui rouvre l'app après une coupure reste bloqué sur des données figées (calendrier en cache, dernier passage périmé) tant qu'il ne rafraîchit pas manuellement. À évaluer : écoute des changements de connectivité (`connectivity_plus`) ou rechargement automatique au retour au premier plan (`WidgetsBindingObserver`/`didChangeAppLifecycleState`) pour redéclencher `charger()` sans action de l'utilisateur.

## Points légaux/confidentialité à traiter avant un lancement public (non bloquant pour le MVP pilote)

- Vérification formelle auprès de la CDP (Commission de Protection des Données Personnelles, Sénégal) sur la nécessité d'une déclaration pour le traitement de géolocalisation.
- Rédaction d'une politique de confidentialité complète (au-delà de la version minimale de la Phase 0).
