# Patterns & conventions — erom-devil (dossier local : erom-agence-devil)

> MàJ : 2026-07-18 (v0.3.0)

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
- code : verdict = table spec + GARDE-FOU (critical security ancrée →
  jamais VALABLE, ne remonte jamais un JETABLE). Ancrage PAR VOIX avant
  consolidation. Tri convergence puis sévérité.

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
- Convention 2026-07-30 : plugin `erom-devil` (= nom du repo GitHub), skills
  et agents SANS préfixe de domaine (`spec`, `brain`, `code`, `*-swarm` ;
  `gemini`, `glm`, `deepseek`, `opus`, `kimi`). Invocation
  `/erom-devil:<skill>`, spawn `erom-devil:<agent>`.
- Les fichiers de `scripts/` gardent le préfixe historique `devil-` :
  `devil-{spec,brain,code}-{mission.md,schema.json}` sont des CHEMINS
  internes, pas des identifiants exposés — ne pas les renommer.
- Feature branch puis merge ff sur main ; commits fréquents avec trailers.
- Tri d'affichage : criticités en français (BLOQUANTE/IMPORTANTE/EXPLORATOIRE),
  enums machine en anglais.

## Exercice code (v0.3.0)
- Le devil juge un CHANGEMENT, jamais un stock : arg chemin → STOP.
- Packaging orchestrateur : DIFF jamais tronqué (> 1 Mo → refus), FILES
  budget 200 Ko (tri par lignes de diff, exclus listés), INTENT jamais
  auto-détecté (.specs interdit — arg explicite ou body PR).
- Scan pré-vol AVANT tout envoi : les fichiers glob-exclus (.env*, *.pem…)
  sont retirés de FILES ET du DIFF (ne partent pas au modèle) ; puis 9 regex
  figées sur le reste (biais faux positif assumé ; résidu acté : secret non
  quoté dans un fichier NON exclu de format non reconnu). Hit → STOP, Romain
  tranche exclure/annuler/forcer.
- Ancrage au retour : file:ligne vs plages de hunks ±3 ; hors périmètre →
  DÉCLASSÉE visible (jamais supprimée), exclue de la correction et du
  garde-fou.
- Correction guidée UNIQUEMENT si le diff reviewé = working tree actuel
  (tree propre requis hors mode working tree).
- Mode working tree : les fichiers non suivis sont inclus en LECTURE SEULE
  (`git diff --no-index /dev/null <f>`, aucun `git add`) ; sinon `git diff
  HEAD` les ignore alors que `git status --porcelain` les compte → feature
  neuve invisible (gap review de branche, corrigé).
- Sévérité `critical` DÉFINIE dans la mission (exploitable/fuite/secret) :
  load-bearing car le garde-fou swarm ne se déclenche que sur
  critical/security ancrée. Ne pas laisser la sévérité au hasard du modèle.
