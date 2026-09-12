# FIXTURES_TEST.md — Jeu de données de test manuel (app chauffeur)

Commandes de management Django (`autombalit-backend`) pour recréer le jeu de données minimal nécessaire au test manuel de l'app chauffeur sur téléphone physique (Tâche 14 : auth OTP + tournée du jour + start/stop ; Tâche 15 : capture GPS/MQTT), après un reset de base ou sur un environnement neuf. Idempotentes — rejouables sans dupliquer les données.

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

## Second jeu de données (créé manuellement, hors commande)

Un premier chauffeur de test (`+221781531736`, numéro réel avec vraie carte SIM) a été créé avant l'existence de ces commandes, avec son propre `Camion` (`DK-0001-TEST`), sa `Tournee` (« Tournée Médina (test manuel) ») et son `AssignationJournaliere` du jour — non recréé par les commandes ci-dessus (noms différents). Pas de commande dédiée pour ce jeu de données ; à recréer manuellement si nécessaire, ou à migrer vers `seed_chauffeur_test`/`seed_tournee_test` avec les bons paramètres si on veut le rendre reproductible aussi.

## Nettoyage

Aucune commande de suppression pour l'instant — ces données de test vivent uniquement en environnement de développement local, jamais en production (pas de garde-fou applicatif empêchant de lancer ces commandes en prod : à ajouter si un jour ces commandes tournaient ailleurs qu'en local).
