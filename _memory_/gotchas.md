# Gotchas — erom-agence-devil

> MàJ : 2026-07-18

## Ollama cloud (contrainte dure, validée Romain 2026-07-18)
- Les modèles `:cloud` (glm-5.2, deepseek-v4-pro) marchent DIRECTEMENT via
  `claude` pointé sur `localhost:11434`. PAS de `ollama pull`, PAS de préflight
  de présence : `/api/tags` ne les liste pas (proxy vers ollama.com) → un
  préflight tags renverrait un faux négatif. Ne jamais re-tester la ligne de base.
- Env : `ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434
  ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max` + `--dangerously-skip-permissions`.

## claude -p headless — hermétisme (flags validés live 2026-07-18)
- `--strict-mcp-config --tools "" --setting-sources "" --no-session-persistence`
  → run hermétique (zéro MCP, zéro tool, pas de settings/CLAUDE.md user parasites).
- `-p --output-format json` : stdout = JSON PUR. La review est dans `.result`.
  Le warning « connectors are disabled » part sur STDERR → toujours capturer
  stdout seul (`2>fichier`).
- Parsing : `jq -r '.result'` puis `sed '/^```/d'` (strip fences éventuelles)
  puis `jq -c`.

## Detail d'erreur : ne PAS remonter le bruit stderr
- Sur échec, le detail diagnostique est `[.api_error_status] + .result` de RAW
  (souvent présents même si `.is_error=true`), PAS le warning « connectors »
  de stderr (bruit constant, trompeur → ferait croire à un pb d'auth au lieu
  d'un modèle absent). Vérifié : modèle bidon → `[404] ... model may not exist`.

## Manifest de plugin Claude Code
- Clé `"agents": string` REJETÉE par le schéma (« agents: Invalid input »).
  Retirer → agents auto-découverts depuis `agents/`. `"skills": string` OK.
- `--plugin-dir` est un flag GLOBAL : il PRÉCÈDE la sous-commande `plugin`
  (`claude --plugin-dir <path> plugin details devil`).

## agy (devil gemini)
- `--print` DERNIER flag avant le prompt (parseur Go consomme le token suivant).
- `< /dev/null` obligatoire (sinon blocage hors TTY).
- Review lue depuis un FICHIER écrit par agy (write_file), jamais stdout
  (bug amont #76 : stdout peut être vide alors que le modèle a répondu).

## Push / remote
- `erom-agence-devil` : origin basculé SSH → HTTPS (SSH publickey denied dans
  cet env ; la marketplace passait déjà en HTTPS). Retour SSH possible :
  `git remote set-url origin git@github.com:eRom/erom-agence-devil.git`.

## Bascule user-level → plugin
- Anciennes versions trashées : `~/.claude/agents/devil-spec-reviewer.md` +
  `~/.claude/skills/devil-spec/` (tuent la collision `/devil-spec`).
- Après push : `claude plugin marketplace update erom-marketplace` PUIS
  `claude plugin install devil@erom-marketplace` (le cache local était obsolète).
- `~/.claude` est un repo git bruité (churn cache plugins) : committer un
  changement ciblé en stageant SEULEMENT le fichier voulu, jamais `git add -A`.
