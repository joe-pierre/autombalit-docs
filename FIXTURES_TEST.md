# FIXTURES_TEST.md — Jeu de données de test manuel (apps chauffeur et citoyen)

Commandes de management Django (`autombalit-backend`) pour recréer le jeu de données minimal nécessaire au test manuel des apps chauffeur et citoyen sur téléphone physique (Tâche 14 : auth OTP + tournée du jour + start/stop ; Tâche 15 : capture GPS/MQTT ; Tâche 19 : écran statut du jour côté citoyen), après un reset de base ou sur un environnement neuf. Idempotentes — rejouables sans dupliquer les données.

À exécuter dans le conteneur backend (`docker compose exec backend ...`), dans cet ordre :

## 1. Chauffeur de test validé

```bash
docker compose exec backend python manage.py seed_chauffeur_test --telephone +221771234567
```

- Crée (ou réutilise) une `Societe` de test (« Société Test Pilote » par défaut, `--societe-nom` pour changer).
- Crée (ou met à jour) un `Chauffeur` avec ce téléphone, `statut_validation=valide`.
- `firebase_uid` volontairement laissé vide : s'attache automatiquement à la première connexion OTP réussie (voir `DECISIONS.md`, Tâche 12).
- `+221771234567` est le numéro de test Firebase configuré côté console (code fixe `123456`, pas de vrai SMS envoyé) — voir Firebase Console → Authentication → Sign-in method → Phone → Phone numbers for testing.

Options : `--nom` (nom du chauffeur), `--societe-nom`.

## 2. Camion + Zone + Tournée + Assignation du jour

```bash
docker compose exec backend python manage.py seed_tournee_test --telephone-chauffeur +221771234567
```

Nécessite que le chauffeur ait déjà été créé par la commande précédente (échoue explicitement sinon, avec le message d'erreur indiquant quelle commande lancer d'abord).

- Réutilise la même `Societe` de test.
- Crée (ou réutilise) un `Camion` (`TEST-001` par défaut, `--immatriculation` pour changer).
- Crée (ou réutilise) une `Zone` de test (polygone factice, sans contour géographique réel — `Zone Test` par défaut, `--nom-zone`).
- Crée (ou réutilise) une `Tournee` de test rattachée à cette zone via `TourneeZone` (`ordre_passage=0`) (`Tournée Test` par défaut, `--nom-tournee`).
- Crée (ou **réinitialise**) l'`AssignationJournaliere` du jour pour ce camion : `chauffeur`/`tournee` remis à jour et `statut` remis à `planifiee` à chaque exécution — pratique pour rejouer un test start/stop complet sans repartir de zéro en base, mais à savoir si un test était en cours (`en_cours`/`terminee`) au moment de relancer la commande, il sera réinitialisé.

Options : `--immatriculation`, `--nom-zone`, `--nom-tournee`, `--nom-societe`.

## 3. Polygone réel pour la Zone de test (test terrain Tâche 19, écran statut du jour côté citoyen)

Le polygone créé par `seed_tournee_test` (étape 2) est un rectangle factice sans contour géographique réel — insuffisant pour tester le rattachement domicile → zone avec un vrai point du monde réel (ex. domicile enregistré depuis l'app citoyen, Tâche 17). Remplace la géométrie de la `Zone` de test par un polygone réel, sans changer son nom/id (donc sans casser la `Tournee`/`TourneeZone` déjà liées) :

```bash
docker compose exec backend python manage.py update_zone_test_polygone zone_ouakam.geojson
```

- `zone_ouakam.geojson` (racine du dépôt `autombalit-backend`) : polygone du quartier d'Ouakam (Dakar), export Google Earth/My Maps (KML converti en GeoJSON — la commande retire automatiquement la 3e coordonnée (Z, généralement 0) que ce type d'export ajoute par point, incompatible avec la colonne PostGIS 2D de `Zone.polygone`).
- Réutilise le pipeline de validation/import GeoJSON déjà validé (`geo_import.services.zone_import`, Tâche 5) : correspondance par `nom` exact (`--nom-zone`, défaut `Zone Test`), donc `Zone.objects.update_or_create` met à jour la géométrie **en place** (id préservé).
- **Recalcule aussi automatiquement `PointEnregistre.zone` pour tous les points déjà enregistrés** (avec la même logique que l'enregistrement initial, `citizens.services.points_enregistres.deduire_zone`) — un point enregistré avant ce changement de géométrie ne se met sinon jamais à jour tout seul (le FK est figé à la création). Sans cette étape, un domicile enregistré avant la mise à jour du polygone resterait bloqué sur « aucune zone connue » côté écran statut du jour, même après la mise à jour de la géométrie.
- Argument positionnel obligatoire : chemin du fichier GeoJSON (`Polygon`, `Feature`, ou `FeatureCollection` avec un seul `Polygon`) ; `--nom-zone` pour cibler une autre zone que `Zone Test`.
- Nécessite que la `Zone` cible existe déjà (échoue explicitement sinon, avec le message indiquant de lancer `seed_tournee_test` d'abord).

## Second jeu de données (créé manuellement, hors commande)

Un premier chauffeur de test (`+221781531736`, numéro réel avec vraie carte SIM) a été créé avant l'existence de ces commandes, avec son propre `Camion` (`DK-0001-TEST`), sa `Tournee` (« Tournée Médina (test manuel) ») et son `AssignationJournaliere` du jour — non recréé par les commandes ci-dessus (noms différents). Pas de commande dédiée pour ce jeu de données ; à recréer manuellement si nécessaire, ou à migrer vers `seed_chauffeur_test`/`seed_tournee_test` avec les bons paramètres si on veut le rendre reproductible aussi.

## Nettoyage

Aucune commande de suppression pour l'instant — ces données de test vivent uniquement en environnement de développement local, jamais en production (pas de garde-fou applicatif empêchant de lancer ces commandes en prod : à ajouter si un jour ces commandes tournaient ailleurs qu'en local).
