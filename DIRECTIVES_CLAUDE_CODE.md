# DIRECTIVES_CLAUDE_CODE.md — Directives prêtes à l'emploi

Ce fichier rassemble les formulations à copier-coller telles quelles dans une session Claude Code, pour les situations récurrentes du projet. Objectif : ne jamais avoir à improviser un prompt, surtout dans l'urgence d'un bug.

Rappel de contexte technique (voir `CLAUDE.md` pour le détail) : trois dépôts séparés (`autombalit-docs`, `autombalit-backend`, `autombalit-mobile`), Claude Code lancé depuis le dossier parent `AutoMbalit/`, aucun commit sans accord explicite.

---

## 1. Démarrer une session (première tâche)

```bash
cd AutoMbalit/
claude
```

Premier message :

```
Lis autombalit-docs/CLAUDE.md et suis le prompt d'amorçage. Puis lis la Tâche [N] dans autombalit-docs/TASK_PROMPTS.md et exécute-la dans le dépôt [autombalit-backend / autombalit-mobile]/, en respectant l'Étape 0 et le Process (pas de commit sans mon accord).
```

Remplacer `[N]` par le numéro de tâche et `[autombalit-backend / autombalit-mobile]` par le dépôt concerné (indiqué dans le champ "Branche" de la tâche).

## 2. Enchaîner sur la tâche suivante (même session)

```
/clear
```

Puis :

```
Exécute la Tâche [N] (TASK_PROMPTS.md, dans autombalit-docs) dans le dépôt [autombalit-backend / autombalit-mobile]/, en respectant l'Étape 0 et le Process (pas de commit sans mon accord).
```

## 3. Demander la relecture avant commit

```
Montre-moi git status et git diff dans [nom du dépôt]/ avant que je décide de committer. N'exécute aucun commit tant que je n'ai pas dit "commit".
```

## 4. Valider le commit

```
commit
```

— committe en local uniquement, dans le dépôt courant.

```
commit et push
```

— committe puis pousse vers GitHub.

Si la tâche prévoit aussi une mise à jour de `TODO.md`/`DECISIONS.md` (prévu en fin de tâche) :

```
Committe aussi la mise à jour de TODO.md et DECISIONS.md dans autombalit-docs, séparément du commit de code.
```

---

## 5. Signaler un bug ou une erreur (template de correction rapide)

À utiliser dès qu'un comportement ne correspond pas à ce qui était attendu — build qui plante, endpoint qui renvoie une erreur, résultat incorrect, etc. Remplir les champs entre crochets avant d'envoyer.

```
Il y a un bug à corriger dans [nom du dépôt]/.

**Contexte :** [ce que tu étais en train de faire — ex. "je testais l'endpoint POST /api/points-enregistres/"]

**Comportement attendu :** [ce qui devrait se passer]

**Comportement observé :** [ce qui se passe réellement]

**Message d'erreur / logs :**
```
[coller ici le message d'erreur, la stack trace, ou les logs pertinents — docker compose logs -f <service> si besoin]
```

**Fichiers ou commande concernés (si connus) :** [chemin de fichier, ou commande exacte qui déclenche le problème]

Diagnostique la cause avant de proposer un correctif — ne suppose rien que tu n'as pas vérifié dans le code ou les logs. Explique la cause en une ou deux phrases, propose le correctif, mais n'applique et ne committe rien tant que je n'ai pas validé.
```

**Pourquoi ce format** : forcer à séparer "attendu" de "observé" évite les corrections qui traitent le symptôme au lieu de la cause — c'est le même principe que ce qu'on a appliqué ensemble sur les bugs du guide (mauvais nom de région Geofabrik, fichier `.env.example` manquant, etc.). La consigne "ne suppose rien que tu n'as pas vérifié" évite qu'il parte sur une fausse piste plausible mais pas vérifiée.

## 6. Une fois le bug corrigé et confirmé

```
Le correctif fonctionne, c'est confirmé. Mets à jour DECISIONS.md dans autombalit-docs (statut "✅ Résolu") et BUGS_AND_ROADMAP.md si c'est un bug de code (pas juste une erreur de manipulation de ma part), puis attends mon accord avant de committer.
```

## 7. Annuler/défaire un changement non commité

```
Annule les modifications non commitées dans [nom du dépôt]/ (git checkout -- . ou git restore, selon ce qui est le plus sûr) et confirme-moi l'état avant de recommencer.
```

⚠️ Ne s'applique qu'aux changements **non commités** — si le mauvais commit est déjà fait, le dire explicitement (`git log --oneline -5` d'abord pour identifier lequel annuler) plutôt que d'utiliser ce template tel quel.

## 8. Décision technique prise en dehors d'une tâche planifiée

Pour les cas comme la découverte "AutoMbalit" dans un texte de rap, ou le choix Docker Compose — une discussion qui débouche sur un choix non prévu dans `TASK_PROMPTS.md` :

```
On vient de prendre une décision technique qui n'était pas prévue dans une tâche : [résumer la décision en une phrase]. Ajoute une entrée dans DECISIONS.md (autombalit-docs) au bon format, committe ce seul fichier, et attends mon accord avant toute autre action.
```

---

## Aide-mémoire rapide

| Situation | Directive |
|---|---|
| Démarrer une tâche | §1 |
| Tâche suivante, même session | §2 |
| Vérifier avant de committer | §3 |
| Committer | §4 |
| Un bug apparaît | §5 |
| Bug corrigé et validé | §6 |
| Annuler un changement non commité | §7 |
| Décision hors tâche planifiée | §8 |
