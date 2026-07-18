# Patterns & conventions — erom-agence-devil

> MàJ : 2026-07-18 (v0.2.0)

## Contrat de spawn (tous exercices → tous agents)
- Prompt : `MISSION_FILE=` / `SCHEMA_FILE=` / `VALIDATE_JQ=` puis `INPUTS:`
  et 1..N lignes `LABEL:chemin_abs`, puis « Exécute la procédure de transport. »
- VALIDATE_JQ = expression `jq -e` de conformité (l'agent la pose en single
  quotes). Testées : spec (prod) et brain (9/9 cas dont bornes).
- Les blocs VALIDATE_JQ doivent rester BYTE-IDENTIQUES entre skill unitaire
  et skill swarm d'un même exercice (risque n°1 de désync des jumelles).

## Enveloppe de sortie (tous les agents)
- Une ligne : `{devil, model, status:"ok", review:{…}}` ou `{devil, model,
  status:"error", error:"CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT", detail}`.
- SCHEMA_INVALID (v0.2.0) : JSON valide mais non conforme ; consomme le retry ;
  le LLM wrapper rédige lui-même les écarts dans detail (machine décide via
  jq -e, LLM explique).
- Échec STRUCTURÉ, jamais un texte d'erreur pris pour une review.

## Synthèses swarm
- spec : verdict ≥2 approve & 0 reject → VALABLE ; ≥2 reject → JETABLE ;
  sinon MODIFS. Issues : convergence 3/3-2/3-1/3, tri convergence puis sévérité.
- brain : PAS de verdict ; consolidation par l'orchestrateur (équivalence =
  même angle mort/même risque), criticité de groupe = max, singletons 1/3,
  tri criticité PUIS convergence. Quorum commun : 3 plein / 2 dégradé annoncé
  / ≤1 échec franc.

## Agents symétriques
- glm est la source ; deepseek = sed `s/glm-5\.2:cloud/deepseek-v4-pro:cloud/g`
  puis `s/glm/deepseek/g` puis `s/GLM/Deepseek/g` (MODÈLE d'abord) +
  contrôle post-gen obligatoire (grep -ci 'glm' = 0, modèle présent).
- gemini à part (protocole agy), non dérivable.

## Robustesse des appels modèle
- Timeout Bash explicite 540000 ms + 1 retry. TMP_DIR unique par run
  (mktemp -d), nettoyage `trash`. Confiance locale assumée (pas de
  sanitization anti-injection, entrées écrites par Romain/Claude).

## Résolution de chemins
- Racine plugin = deux niveaux au-dessus du base dir injecté dans la skill.
  Tout en absolu. Zéro `~/.claude/` en dur (grep de contrôle).

## Exécution de chantier (méthode validée v0.2.0)
- Subagent-driven : brief extrait par tâche (`task-brief`), implémenteur
  sonnet (extraction sed byte-à-byte des blocs 4-backticks du brief, jamais
  retaper), reviewer sonnet par tâche (2 verdicts : spec + qualité), review
  finale de branche sur opus, ledger .superpowers/sdd/progress.md.
- Pré-vol : revue adversariale du plan AVANT Task 1 (a payé : 1 Important).
- Devils en test : spawn general-purpose « lis agents/X.md et exécute » +
  fallback fichier scratchpad (canal teammate intermittent).

## Nommage & git
- Exercices préfixés : `devil-spec-*` / `devil-brain-*` pour missions,
  schémas, skills ; agents SANS exercice (`devil-<modèle>`).
- Feature branch puis merge ff sur main ; commits fréquents avec trailers.
- Tri d'affichage : criticités en français (BLOQUANTE/IMPORTANTE/EXPLORATOIRE),
  enums machine en anglais.
