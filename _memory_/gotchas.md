# Gotchas — erom-devil (dossier local : erom-agence-devil)

> MàJ : 2026-08-20 (v0.7.0)

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
- Re-confirmé le 2026-08-20 au prix d'un run de devil : un `Agent` nommé
  `devil-deepseek` a produit une review complète et une réponse détaillée à
  ma question de diagnostic. AUCUN des deux ne m'est parvenu, seulement
  trois `idle_notification`. Redemander ne sert à rien, c'est un défaut
  d'outillage. Récupération par le transcript frère
  `~/.claude/projects/<projet>/<sessionId>.jsonl` (le plus récent après
  celui de la session mère), en extrayant les `tool_use` Bash et les blocs
  texte avec `jq`. Cela a parfaitement marché.
- `shutdown_request` via SendMessage retourne « success » et ne tue PAS un
  teammate in-process : il a re-signalé idle juste après. `TaskStop` avec le
  NOM de l'agent en `task_id` l'a arrêté du premier coup
  (`task_type: in_process_teammate`). Utiliser TaskStop, pas le protocole
  de shutdown.

## Le timeout de 540 s des agents est trop court pour l'effort max
- Rejouable : `tests-devil/` (paquet figé de 128 338 octets, deux scripts).
- Mesure du 2026-08-20, même paquet, même modèle
  `deepseek-v4-pro:cloud[1m]`, même ligne d'appel, seule l'effort change :

  | | `max` | `low` |
  |---|---|---|
  | durée API | **620 s** | 90 s |
  | `stop_reason` | `end_turn` | `end_turn` |
  | `result` | 11 610 caractères | 5 667 caractères |
  | tokens de sortie | **115 018** | 15 683 |
  | coût | 3,90 USD | 0,79 USD |
  | verdict rendu | 72, `rework`, **6 issues** | 84, `approve`, 2 issues |

- **La cause est le timeout de la procédure de transport, pas l'effort.**
  Les `agents/*.md` codaient `timeout` Bash à 540 000 ms ; à max, ce paquet
  demande 620 s. Le devil était tué 80 s avant la fin. C'est Romain qui l'a
  trouvé en passant à 1200 s, après une soirée où je l'avais pris pour une
  constante intouchable.
- **Monter le nombre dans les agents NE SUFFIT PAS** (doc Claude Code,
  « Timeout and output limits », validée le 2026-08-20) : `timeout` est
  une DEMANDE, plafonnée par `BASH_MAX_TIMEOUT_MS`, dont le défaut est
  **10 minutes**. Les agents demandent 1 200 000 ms depuis ce soir, mais
  tant que `BASH_MAX_TIMEOUT_MS` n'est pas relevé, tout run de plus de
  600 s est tué. Le régler dans le bloc `env` d'un `settings.json`, ou
  dans l'environnement de la session qui invoque la skill.
  `BASH_DEFAULT_TIMEOUT_MS` (2 min) est l'autre borne, celle qui
  s'applique quand aucun timeout n'est passé ; le plafond effectif est le
  plus grand des deux.
- Piège de validation, vécu le soir même : un run de contrôle a rendu en
  345 s et j'ai failli en conclure que le plafond était levé. Il ne
  prouvait rien, puisqu'il n'a jamais atteint 600 s. La variabilité est
  large sur ce modèle (345 s et 620 s pour le même paquet) : un run court
  ne teste pas une limite haute.
- **À max la critique est franchement meilleure**, ce qui inverse
  l'arbitrage : 6 issues contre 2, dont une `high` réelle (asymétrie entre
  la borne de sévérité de l'Étape 8 et le périmètre catégoriel de
  l'Étape 9) que le run à low n'a pas vue. Pour une porte de merge payée
  une fois avant merge, 10 minutes et 3,90 USD sont le bon prix.
- **Incident distinct, non reproduit** : un run à max le même soir s'est
  arrêté à 194 s avec `stop_reason: stop_sequence`, `.result` vide,
  `usage.output_tokens: 0` et 0,54 USD facturés, sans être tué par le
  timeout (exit 0). Ce n'est PAS le comportement normal à max. Si ça
  revient, le symptôme à chercher est un `.result` vide avec
  `is_error: false` : un transport qui ne teste que `is_error` ou « RAW
  est-il vide » ne voit rien passer, et sort en `PARSE_ERROR` à l'étape
  suivante.
- Leçon de méthode, la plus chère de la soirée : j'ai construit tout un
  diagnostic en tenant le timeout de 540 s pour une donnée du problème,
  parce qu'il était écrit dans la procédure. Quand une mesure bute sur une
  limite, tester la limite AVANT d'expliquer ce qu'il y a derrière.
- Ce n'est ni le modèle ni le volume seuls : à effort max, un prompt
  minimal répond en **719 ms** (`is_error:false`).
- **L'effort atteint réellement le modèle : chaîne vérifiée de bout en
  bout le 2026-08-20.** Deux preuves indépendantes, après un faux
  soupçon (voir plus bas) :
  1. **Ce qui part sur le fil**, capturé en pointant `ANTHROPIC_BASE_URL`
     vers un serveur local qui logge le body : `CLAUDE_CODE_EFFORT_LEVEL=max`
     et le flag `--effort max` produisent un body IDENTIQUE, portant
     `output_config: {effort: "max"}` (contre `"high"` sans rien, l'effort
     de la session parente). La variable d'environnement n'a rien
     d'inerte, et le flag ne fait rien de plus qu'elle.
  2. **Ce qu'Ollama en fait**, lu dans `anthropic/anthropic.go:378-400` :
     `output_config.effort` est traduit en `api.ThinkValue`, et `"max"`
     figure dans la liste acceptée (`high`, `medium`, `low`, `max`). La
     priorité irait au champ `thinking` s'il valait `enabled` ou
     `disabled` ; or le CLI envoie `{"type":"adaptive"}`, qui ne matche ni
     l'un ni l'autre, donc l'effort n'est jamais écrasé.
- **Piège de diagnostic n°2, celui qui m'a fait dérailler** :
  `thinking_tokens` vaut 0 dans la réponse quel que soit l'effort, y
  compris avec le flag officiel. Ce n'est PAS un signe que l'effort se
  perd, seulement que ce modèle ne rapporte pas ses tokens de réflexion
  via ce transport. Ne jamais s'en servir comme marqueur d'effort ici :
  capturer le body sortant, c'est cinq minutes et ça ne ment pas.
- **`xhigh` est silencieusement rabattu sur `high`** par Ollama
  (`anthropic.go:384`). Sur les transports ollama, demander xhigh donne
  high. Seul `max` monte réellement au-dessus.
- Ce qui reste non reproduit : le run « complet + max » est celui du
  subagent devil, pas une mesure directe. L'explication est solide, la
  seconde moitié de la mesure reste à refaire en direct.
- Le commit fa9ff98 (2026-08-20, Romain) a ajouté `--effort max` en flag
  aux trois agents ollama, en plus de la variable déjà présente. Redondant
  et inoffensif : les deux écrivent le même champ.
- **Ce qui est mesuré, et par qui.** Les deux runs à effort low et le run
  minimal à max sont des mesures directes de la session mère. Le run
  « complet + max » vient du subagent devil, récupéré dans son transcript
  et NON reproduit en direct : ses deux tentatives appartiennent au même
  run, ce ne sont pas deux mesures indépendantes, et son TMP_DIR a été
  détruit par le `trash` de son étape 4. Reproduire en direct avant de
  traiter ce point comme définitif.
- **Non mesuré** : le seuil de bascule entre un prompt minimal et 101 Ko,
  et le comportement des autres modèles. Seul `deepseek-v4-pro:cloud[1m]`
  a été testé ; l'extrapolation aux 5 agents repose sur le seul fait
  qu'ils posent tous `CLAUDE_CODE_EFFORT_LEVEL=max` en dur dans leur
  Step 2, pas sur une mesure.
- Conséquence pratique, dans la limite de ce qui précède : sur un paquet
  de l'ordre de 100 Ko, la porte de merge ne rend pas. Non corrigé :
  l'arbitrage qualité contre latence appartient à Romain (baisser l'effort
  dégrade la critique, allonger le timeout allonge l'attente).
- Re-jouer la mesure : reconstruire le prompt hermétique, lancer deux fois
  la ligne de `agents/deepseek.md` Step 2 en ne changeant que l'effort.
- Piège de diagnostic : le stderr porte
  `[claude-code:unrecognized_model] {"model":"deepseek-v4-pro:cloud[1m]",
  "query_source":"generate_session_title"}` **même quand l'appel
  réussit**. C'est cosmétique (génération du titre de session), ce n'est
  jamais la cause. Vérifié sur un appel minimal `is_error:false`.

## Push / remote
- Les 2 repos en HTTPS (SSH publickey denied dans cet env). Marketplace :
  entrée devil à bump (version + description) EN PLUS de metadata.version.
