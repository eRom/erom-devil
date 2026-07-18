# Gotchas — erom-agence-devil

> MàJ : 2026-07-18 (v0.2.0)

## Ollama cloud (contrainte dure, validée Romain 2026-07-18)
- Les modèles `:cloud` (glm-5.2, deepseek-v4-pro) marchent DIRECTEMENT via
  `claude` pointé sur `localhost:11434`. PAS de `ollama pull`, PAS de préflight
  de présence : `/api/tags` ne les liste pas (proxy vers ollama.com) → un
  préflight tags renverrait un faux négatif. Ne jamais re-tester la ligne de base.
- Env : `ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434
  ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max` + `--dangerously-skip-permissions`.

## claude -p headless — hermétisme (flags validés live)
- `--strict-mcp-config --tools "" --setting-sources "" --no-session-persistence`
  → run hermétique. `-p --output-format json` : stdout = JSON PUR, review dans
  `.result`. Warning « connectors are disabled » = bruit STDERR constant.
- Parsing : `jq -r '.result'` puis `sed '/^```/d'` puis `jq -c`.

## Detail d'erreur : ne PAS remonter le bruit stderr
- Sur échec, detail = `[.api_error_status] + .result` de RAW, PAS stderr.
  Vérifié (v0.1.0 ET re-vérifié v0.2.0) : modèle bidon → `[404] ... may not exist`.

## Mise à jour d'un plugin installé (appris à la livraison v0.2.0)
- `claude plugin install <déjà-installé>` = NO-OP (« already installed »),
  ne re-tire PAS le cache. `claude plugin update devil` (nom nu) ÉCHOUE
  (« not found »). La voie fiable : `claude plugin uninstall devil` PUIS
  `claude plugin install devil@erom-marketplace` (après `marketplace update`).
- Les nouvelles skills/agents n'apparaissent dans une session déjà ouverte
  qu'après REDÉMARRAGE de la session.

## Manifest de plugin Claude Code
- Clé `"agents": string` REJETÉE par le schéma. Retirer → auto-découverte
  depuis `agents/`. `--plugin-dir` est un flag GLOBAL (précède la sous-commande).

## agy (devil gemini)
- `--print` DERNIER flag avant le prompt ; `< /dev/null` obligatoire ;
  review lue depuis un FICHIER écrit par agy (bug amont #76, stdout parfois vide).

## sed glm→deepseek
- MODÈLE d'abord (`glm-5\.2:cloud` avant `glm`), sinon chimère
  `deepseek-5.2:cloud`. Contrôle post-gen OBLIGATOIRE : `grep -ci 'glm'` = 0
  (exit 1 = succès) + présence `deepseek-v4-pro:cloud`. Note : `Glob` ne
  contient pas `glm` (g-l-o-b), pas de faux positif.

## Greps de non-présence
- exit 1 = SUCCÈS attendu (rien trouvé). Ne jamais partir en fix sur ce code
  retour ; lire la sortie. (Piège récurrent pour agents/wrappers d'exécution.)

## Hook rtk local
- Réécrit les sorties git/grep/ls même parfois avec `command` (ex. plage de
  commits condensée) → pour un hash/état décisionnel, vérifier via
  `git rev-parse` ciblé ou redirection fichier + Read. Un implémenteur a
  rapporté une plage fausse (70afb04..) à cause de ça ; le commit réel était propre.

## Teammates / agents nommés
- Le canal de retour des agents NOMMÉS (mode teammate) est intermittent :
  idle notification sans livrable = à traiter (redemander + fallback fichier
  scratchpad). Les agents SANS nom retournent leur résultat de façon fiable
  via task-notification. Pour du fan-out fiable : agents anonymes + fichier.

## Push / remote
- Les 2 repos en HTTPS (SSH publickey denied dans cet env). Marketplace :
  entrée devil à bump (version + description) EN PLUS de metadata.version.
