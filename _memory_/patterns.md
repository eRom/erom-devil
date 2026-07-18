# Patterns & conventions — erom-agence-devil

> MàJ : 2026-07-18

## Contrat de sortie (tous les devils)
- Enveloppe une ligne : `{devil, model, status:"ok", review:{…}}` ou
  `{devil, model, status:"error", error:"CLI_FAILED|PARSE_ERROR|TIMEOUT", detail}`.
- L'échec est STRUCTURÉ et reconnaissable : jamais un texte d'erreur pris pour
  une review (leçon playbook sur les retours de sous-agents).
- Verdict 3 états : approve ≥ 80 / rework 50-79 / reject < 50.

## Synthèse swarm (orchestrateur)
- Verdict : ≥ 2 approve & 0 reject → VALABLE ; ≥ 2 reject → JETABLE ; sinon →
  MODIFICATIONS REQUISES. Calcul sur les VOIX EXPRIMÉES (tolère 2 voix).
- Convergence des issues : regroupement sémantique (pas textuel), badge
  3/3-2/3-1/3, tri convergence puis sévérité.
- Voix dissonante jamais écrasée : section dédiée si un devil s'écarte.

## Agents symétriques
- glm est la source ; deepseek = `sed -e 's/glm-5\.2:cloud/deepseek-v4-pro:cloud/g'
  -e 's/glm/deepseek/g' -e 's/GLM/Deepseek/g'` (MODÈLE d'abord, sinon corruption
  `deepseek-5.2:cloud`).
- gemini est à part (protocole agy), pas dérivable par sed.

## Robustesse des appels modèle
- Timeout Bash explicite **540000** ms (9 min) + **1 retry** sur tout appel.
- Prompt assemblé dans un fichier tmp ; nettoyage via `trash` (jamais `rm`).

## Résolution de chemins
- Racine plugin = deux niveaux au-dessus du « Base directory » injecté dans la
  skill. SCHEMA_FILE / MISSION_FILE passés en ABSOLU aux agents.
- Zéro chemin `~/.claude/` en dur (vérifié par grep en fin de chaque skill).

## Nommage & git
- Préfixe `devil-spec-*` : laisse la place à un futur `devil-code-*`.
- Commits fréquents (1 par task TDD), trailers Co-Authored-By + Claude-Session.
- Feature branch puis merge ff sur main (repo greenfield).

## Vérification (pas de suite de tests classique)
- Smoke live par devil sur les fixtures + test du chemin d'erreur (modèle bidon)
  + `claude --plugin-dir . plugin details devil` (inventaire) + dogfood swarm.
