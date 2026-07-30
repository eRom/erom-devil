# Gotchas — erom-devil (dossier local : erom-agence-devil)

> MàJ : 2026-07-18 (v0.3.0)

## Masquage credentials = AFFICHAGE seulement (piège de transcription v0.3.0)
- Le hook PII local réécrit les credentials (clé AWS `AKIAIOSFODNN7EXAMPLE`
  → `[REDACTED:aws_key]`) dans TOUT ce qui s'affiche (tool results, diffs,
  lectures) — mais le DISQUE est intact. Un `Write` de la vraie clé la stocke
  littéralement ; `grep -E 'AKIA[0-9A-Z]{16}'` matche (exit 0) même si la
  ligne s'affiche masquée.
- Conséquence : NE JAMAIS juger la présence d'un secret/clé à l'œil. Vérifier
  par `grep -c -F`, `grep -E -c`, ou `od -c`. Un plan/brief peut contenir la
  vraie clé sur disque tout en s'affichant `[REDACTED]` (les fixtures scan
  du plugin en dépendent). Piège attrapé par la revue adversariale du plan.

## Working tree : untracked invisibles à `git diff HEAD` (gap review v0.3.0)
- `git status --porcelain` COMPTE les fichiers non suivis (`??`), mais
  `git diff HEAD` les EXCLUT. Une feature de fichiers neufs non ajoutés →
  détectée « sale » mais diff vide → faux « rien à reviewer » ; changement
  mixte → fichiers neufs droppés en silence.
- Fix skill `code` : inclure les untracked en LECTURE SEULE via
  `git diff --no-index /dev/null <f>` (aucun `git add`, aucune mutation
  d'index). Cette commande sort en CODE 1 dès qu'elle trouve une différence
  (comparaison à /dev/null) — c'est normal, pas un échec.

## jq `all(cond)` sur un tableau (réfutation d'une review devil)
- `.issues | all(has("x") and …)` itère bien sur les ÉLÉMENTS du tableau et
  vaut `true` si tous satisfont la condition (validé jq 1.8.1). Un devil a
  affirmé cette forme « syntaxiquement invalide » — FAUX, réfuté par test.
  Toujours vérifier une accusation de syntaxe jq par exécution avant de la
  traiter comme un finding.

## Suffixe `[1m]` : TOUJOURS quoter (bug silencieux 2026-07-24 → 07-29)
- Le Bash tool tourne sous `/bin/zsh`, où l'option `nomatch` est active :
  `[1m]` non quoté est lu comme une CLASSE DE CARACTÈRES glob. `claude
  --model glm-5.2:cloud[1m]` → `zsh: no matches found` et la commande n'est
  JAMAIS exécutée. En bash le pattern passerait littéralement — d'où un bug
  invisible pour qui teste hors harnais.
- Symptôme trompeur : en substitution `RAW=$(… )`, la subshell sort en 1,
  `RAW` est vide, le transport conclut `CLI_FAILED` avec un detail de panne
  réseau alors qu'AUCUN appel modèle n'a eu lieu. Le retry échoue pareil.
- Introduit par le commit `ab4886f` (ajout du `[1m]`), il a cassé glm ET
  deepseek pendant 5 jours sans que rien ne le signale. Forme correcte :
  `claude --model "glm-5.2:cloud[1m]"`. Vérif : le `modelUsage` du JSON de
  sortie doit contenir le tag AVEC son suffixe.
- Sans danger dans une valeur JSON de programme `jq` (`model:"…[1m]"`,
  testé exit 0) : le piège est le shell, pas jq.

## Kimi (kimi-k3:cloud) — hors forfait Ollama
- Tag exact `kimi-k3:cloud` (2.8T params, MXFP4, `context_length` 1048576 →
  le `[1m]` est justifié). `kimi-3:cloud` et `kimi-k2:cloud` n'existent pas ;
  `kimi-k2-thinking:cloud` répond 410 (retiré).
- Double suffixe = 400 : `API Error: 400 use either :local or :cloud, not
  both`. Piège classique du sed de clonage glm→kimi.
- Le modèle est en **extra usage only**, PAS inclus dans le forfait : sans
  solde, `402 … extra usage balance is empty` (crédit sur
  ollama.com/settings). Le transport est correct, le refus vient du compte —
  ne pas partir en debug de la ligne d'appel sur ce code.

## Ollama cloud (contrainte dure, validée Romain 2026-07-18)
- Les modèles `:cloud` (glm-5.2, deepseek-v4-pro) marchent DIRECTEMENT via
  `claude` pointé sur `localhost:11434`. PAS de `ollama pull`, PAS de préflight
  de présence : `/api/tags` ne les liste pas (proxy vers ollama.com) → un
  préflight tags renverrait un faux négatif. Ne jamais re-tester la ligne de base.
- Env : `ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434
  ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max` + `--dangerously-skip-permissions`.

## Devils externes — confidentialité (recadrage Romain 2026-07-18)
- L'envoi des specs/brainstorms aux devils (Gemini via agy, Ollama cloud) est
  le design même du plugin : AUCUN avertissement de confidentialité à émettre
  (« c'est implicite »). Ne pas re-proposer de garde-fou là-dessus.

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
- RÉSOLU 2026-07-18 : exclusions actives via `~/Library/Application Support/
  rtk/config.toml` (convention Apple, symlink vers `~/.config/rtk/config.toml`)
  — grep, curl, find, git log, git status, git branch passent en clair.
  `ls` et le reste restent réécrits. Le préfixe `command` ne protège PAS
  (réécriture par hook harnais, avant le shell). Historique : plage de commits
  faussée (70afb04..) quand git log était encore réécrit ; le commit réel
  était propre.

## Teammates / agents nommés
- Le canal de retour des agents NOMMÉS (mode teammate) est intermittent :
  idle notification sans livrable = à traiter (redemander + fallback fichier
  scratchpad). Les agents SANS nom retournent leur résultat de façon fiable
  via task-notification. Pour du fan-out fiable : agents anonymes + fichier.

## Push / remote
- Les 2 repos en HTTPS (SSH publickey denied dans cet env). Marketplace :
  entrée devil à bump (version + description) EN PLUS de metadata.version.
