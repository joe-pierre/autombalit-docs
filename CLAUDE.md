# CLAUDE.md — Contexte et prompt d'amorçage

## Résumé du projet

**AutoMbalit** : application de géolocalisation des camions de collecte d'ordures au Sénégal. Remplace le klaxon des camions par des notifications basées sur l'ETA (30/20/10/5 min). Trois composants : app Flutter citoyen, app Flutter chauffeur, web admin (SIG) — le tout piloté par un backend Django + GeoDjango.

**Stack** : Django + DRF + GeoDjango, PostgreSQL/PostGIS, OSRM self-hosted, MQTT (Mosquitto), Django Channels + Redis, Firebase (Auth + FCM), JWT, Flutter (`flutter_map`/OSM). Détail complet dans `SPEC.md` §2. Environnement local géré via Docker Compose (voir `GUIDE_DU_DEVELOPPEUR.md`), tests mobiles sur téléphone Android physique en USB (pas d'émulateur).

**Organisation des dépôts** : ce fichier et les autres `.md` de suivi vivent dans un dépôt séparé, `autombalit-docs`, distinct de `autombalit-backend` et `autombalit-mobile`. **Important** : Claude Code ne charge PAS automatiquement ce fichier — il remonte l'arborescence depuis le dossier de lancement vers le haut, jamais vers le bas dans un sous-dossier comme `autombalit-docs/`. Il faut donc explicitement dire, en premier message de chaque session : *"Lis autombalit-docs/CLAUDE.md et suis le prompt d'amorçage"*.

**Option recommandée — lancer depuis le dossier parent (`AutoMbalit/`)** :
```bash
cd AutoMbalit/
claude
```
Les trois sous-dossiers (`autombalit-docs`, `autombalit-backend`, `autombalit-mobile`) sont accessibles sans configuration supplémentaire — aucun `--add-dir` nécessaire puisqu'ils sont tous sous-dossiers du dossier de lancement. Permet d'enchaîner plusieurs tâches touchant des dépôts différents dans la même session, en précisant à chaque fois le dossier cible (ex. *"Exécute la Tâche 2 dans autombalit-backend/"*). Seul point de vigilance : `AutoMbalit/` n'est pas lui-même un dépôt Git — toute commande `git` doit explicitement cibler le bon sous-dossier, sinon elle échoue ("not a git repository").

**Option alternative — lancer depuis l'intérieur d'un dépôt** :
```bash
cd autombalit-backend
claude --add-dir ../autombalit-docs
```
Git est alors directement scopé au bon dépôt sans avoir à le préciser, mais `--add-dir` est nécessaire pour lire `autombalit-docs` (devenu un dossier voisin), et il faut relancer une nouvelle session si on change de dépôt cible (ex. passer à `autombalit-mobile`).

**Mécanique Git multi-dépôts** : chaque commande `git` s'applique au dépôt le plus proche dans l'arborescence (celui qui contient le `.git`) — toujours s'assurer d'être dans le bon dépôt avant de committer. Une tâche qui touche à la fois du code et le suivi (`TODO.md`/`DECISIONS.md`) produit **deux commits séparés, dans deux dépôts différents** : un commit de code dans `autombalit-backend` ou `autombalit-mobile`, un commit de documentation dans `autombalit-docs`. Ne jamais fusionner les deux en un seul commit. "commit" seul = commit local uniquement ; dire explicitement "commit et push" pour pousser vers GitHub dans la foulée.

**Règle d'or du projet** : pas de code écrit à la main directement dans le projet — le développement passe par des prompts dédiés donnés à Claude Code, listés dans `TASK_PROMPTS.md`, un par un, avec accord explicite avant tout commit. Les formulations exactes à utiliser (démarrage de tâche, signalement de bug, commit, annulation) sont dans `DIRECTIVES_CLAUDE_CODE.md` — s'y référer plutôt que d'improviser un prompt.

## Prompt d'amorçage (à exécuter au début de chaque session)

Au début de chaque session, lire dans cet ordre :
1. `SPEC.md` — spécifications fonctionnelles et techniques
2. `CONVENTIONS.md` — règles de codage, dont l'architecture en couches (service layer, anti god-file — voir §Architecture et paradigmes)
3. `GUIDE_DU_DEVELOPPEUR.md` — comment l'environnement local et le déploiement sont censés fonctionner
4. `TODO.md` — état d'avancement des tâches
5. `DECISIONS.md` — décisions techniques et bugs résolus
6. `CODE_SNAPSHOT.md` — snapshot du code actuel (s'il existe)

Puis résumer en 5 points :
- Ce que fait le projet
- La stack technique utilisée
- L'état d'avancement actuel
- Les conventions importantes à respecter
- Les décisions clés déjà prises

À la fin de chaque tâche :
- Mettre à jour `DECISIONS.md` si une décision technique a été prise ou un bug résolu
- Mettre à jour `TODO.md` pour cocher les tâches accomplies et ajouter les suivantes
- Mettre à jour `BUGS_AND_ROADMAP.md` si un bug a été corrigé ou une idée identifiée
- Ne rien committer sans instruction explicite ("commit")

---

## Formats des fichiers Markdown modifiables

### Format `TODO.md`

````markdown
# TODO

## Phase X — Nom de la phase
- [ ] Tâche à faire
- [x] Tâche accomplie
````

### Format `DECISIONS.md`

````markdown
## [RÉSOLU|CHOIX] Titre de la décision

**Contexte :** ...
**Symptôme / Problème :** ... (si bug)
**Cause / Alternatives :** ...
**Fix / Décision :** ...
**Leçon :** ...
**Statut :** ✅ Résolu | 🔵 Choix assumé
````

### Format `TASK_PROMPTS.md`

Chaque tâche suit la structure : Contexte, Branche, Constat, Étape 0 — Avant de coder, TÂCHE (sous-tâches numérotées + exclusions explicites), Contraintes, Process, Critère d'acceptation. Voir les tâches déjà rédigées dans `TASK_PROMPTS.md` comme référence de format.
