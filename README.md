# AutoMbalit — Fichiers du workflow

Application de géolocalisation des camions de collecte d'ordures au Sénégal (voir `SPEC.md` §1 pour la vue d'ensemble complète).

## Fichiers

- `CLAUDE.md` : contexte projet + prompt d'amorçage pour chaque session (Claude Code ou autre).
- `SPEC.md` : spécifications fonctionnelles et techniques (modèle de données, règles métier, architecture, identité visuelle...).
- `CONVENTIONS.md` : règles de codage (nommage, architecture en couches, sécurité, tests).
- `GUIDE_DU_DEVELOPPEUR.md` : guide pas à pas de A à Z — environnement local (Docker Compose + test sur téléphone physique), déploiement VPS, publication mobile, utilisation finale.
- `TASK_PROMPTS.md` : prompts de tâches prêts à donner à Claude Code (backend et mobile).
- `TODO.md` : suivi des tâches, par phase.
- `DECISIONS.md` : journal des décisions techniques et bugs résolus.
- `BUGS_AND_ROADMAP.md` : bugs corrigés et idées pour les versions futures.
- `CODE_SNAPSHOT.md` : snapshot de code (généré automatiquement, absent tant qu'aucun code n'existe).

## Ordre de lecture recommandé

1. `SPEC.md` — comprendre le projet et les règles métier avant tout.
2. `CONVENTIONS.md` — comment coder une fois qu'on sait quoi coder.
3. `GUIDE_DU_DEVELOPPEUR.md` — comment faire tourner et déployer concrètement.
4. `TODO.md` + `DECISIONS.md` — où on en est et pourquoi.
5. `TASK_PROMPTS.md` — la tâche à exécuter maintenant.

Voir `CLAUDE.md` pour le prompt d'amorçage complet à utiliser en début de session.
