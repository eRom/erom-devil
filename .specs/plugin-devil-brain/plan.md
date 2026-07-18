# devil brain v0.2.0 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ajouter l'interrogatoire socratique `/devil-brain` + `/devil-brain-swarm` au plugin devil, en généralisant les 3 agents en transport pur partagés avec `/devil-spec`.

**Architecture:** Les agents deviennent des transporteurs sans sémantique d'exercice (mission + schéma + validation + inputs étiquetés fournis par les skills). Les skills portent l'exercice : spec passe 2 fichiers et le contrat spec, brain passe 1 fichier et le contrat brain (0 à 5 questions, sans score ni verdict). Référence : `.specs/plugin-devil-brain/{brainstorming,architecture-technique}.md` (post-tribunal, commit f444f4f).

**Tech Stack:** Markdown (agents + skills Claude Code plugin), bash + `jq` + `sed`, `agy` (Gemini), `claude -p` → ollama cloud (GLM/Deepseek), `trash`.

## Global Constraints

- Jamais `rm` / `rmdir` : suppression via `trash` uniquement.
- Timeout Bash explicite **540000** ms + **1 retry** sur tout appel modèle.
- Jamais `ollama pull`, jamais de préflight `/api/tags` (les modèles `:cloud` n'y figurent pas, c'est normal et validé).
- agy : `--print` DERNIER flag avant le prompt, `< /dev/null` obligatoire, review lue depuis le FICHIER écrit par agy (bug stdout #76).
- claude -p : env `ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434 ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max`, flags `--dangerously-skip-permissions --strict-mcp-config --tools "" --setting-sources "" --no-session-persistence -p --output-format json` ; review dans `.result` ; le warning stderr « connectors are disabled » est du bruit constant, jamais un diagnostic.
- Detail d'erreur ollama : `[.api_error_status] + .result` de RAW, stderr en dernier recours seulement.
- Manifest plugin : PAS de clé `agents` (rejetée par le schéma, auto-découverte depuis `agents/`) ; `--plugin-dir` est un flag GLOBAL (avant la sous-commande `plugin`).
- Enums JSON en anglais, contenus rédigés en français.
- Zéro chemin `~/.claude/` en dur dans `skills/` et `agents/`.
- Surfaces `/devil-spec` et `/devil-spec-swarm` : syntaxe, verdicts, seuils, synthèse et boucle de correction INCHANGÉS.
- Sorties de commande pilotant une décision : préfixer `command` (hook rtk réécrit git/grep/ls).
- Commits : trailers `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` + `Claude-Session: https://claude.ai/code/session_013zG81zFmmGeLihXvXSxQ6u`.

## Contrat de spawn agent (interface commune, référencée par les tâches 2 à 6 et 8 à 9)

Prompt de spawn d'un agent devil (texte, lignes exactes) :

```
MISSION_FILE=<abs>
SCHEMA_FILE=<abs>
VALIDATE_JQ=<expression jq -e sur une ligne>
INPUTS:
<LABEL>:<abs>
[<LABEL>:<abs>]

Exécute la procédure de transport.
```

Expressions VALIDATE_JQ par exercice (l'agent la pose en bash entre single quotes) :

- spec (éprouvée v0.1.0) : `has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.verdict | IN("approve","rework","reject"))`
- brain (validée 8/8 cas le 2026-07-18, incluant tableau vide, 6 questions, enum inconnue, champ manquant, non-tableau) : `has("assessment") and has("questions") and (.questions|type=="array" and length<=5) and (.questions|all(has("question") and has("domain") and has("risk") and (.criticality|IN("blocking","important","exploratory"))))`

INPUTS par exercice : spec → `BRAINSTORMING:<abs>` + `SPECS:<abs>` ; brain → `BRAINSTORMING:<abs>` seul.

Enveloppe de retour (contrat strict, une ligne) : `{"devil":"<d>","model":"<m>","status":"ok","review":{…}}` ou `{"devil":"<d>","model":"<m>","status":"error","error":"CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT","detail":"≤ 500 chars"}`.

---

### Task 1: Contrats par exercice (scripts/)

**Files:**
- Rename: `scripts/devil-mission.md` → `scripts/devil-spec-mission.md` (contenu inchangé)
- Rename: `scripts/spec-review-schema.json` → `scripts/devil-spec-schema.json` (contenu inchangé)
- Create: `scripts/devil-brain-mission.md`
- Create: `scripts/devil-brain-schema.json`

**Interfaces:**
- Produces: les 4 chemins ci-dessus, consommés par les skills (Tasks 4, 5, 6) via `<racine plugin>/scripts/<fichier>`.

- [ ] **Step 1: Renommer les contrats spec**

```bash
cd /Users/recarnot/dev/erom-agence-devil
command git mv scripts/devil-mission.md scripts/devil-spec-mission.md
command git mv scripts/spec-review-schema.json scripts/devil-spec-schema.json
```

- [ ] **Step 2: Créer `scripts/devil-brain-mission.md`** (contenu complet)

```markdown
Tu es un architecte logiciel senior en sparring partner socratique. On te
donne UN document : un BRAINSTORMING (définition de besoins d'un projet).
Ton travail n'est PAS de juger ni de noter ce document. Ton travail est de
trouver ce qui n'a JAMAIS été pensé : les questions structurantes que ce
document n'a jamais posées.

Rends les questions les PLUS DANGEREUSES — 5 maximum, en français. Une
question est dangereuse si écrire une spécification sans y répondre crée un
risque réel et nommable.

Grille de domaines, à balayer MENTALEMENT uniquement (aide-mémoire, jamais
un questionnaire) : périmètre et non-buts, utilisateurs et parcours,
données et cycle de vie, sécurité et accès, erreurs et modes dégradés,
exploitation/ops, coûts et limites, dépendances externes, réglementaire.

Interdictions dures :
- JAMAIS réciter la grille : une question par case cochée = échec total.
- Une question n'est posable QUE si elle est spécifique à CE projet et que
  l'absence de réponse crée un risque réel, que tu nommes dans `risk`.
- Une question dont la réponse figure déjà dans le document = faute grave.
  Relis le document avant de rendre chaque question.
- Aucune question générique valable pour n'importe quel projet.
- 0 question est une sortie légitime et respectable si le document est
  solide : ne remplis JAMAIS le quota pour faire bonne figure.

Champs de sortie :
- `assessment` : ton impression générale en UNE ligne (du contexte, pas un
  verdict ni une note).
- `question` : formulée pour être posée telle quelle au porteur du projet.
- `domain` : le domaine concerné, formulation libre.
- `risk` : ce qui se passe concrètement si on écrit la spec sans la réponse.
- `criticality` : `blocking` (spécifier sans répondre serait dangereux),
  `important` (zone de flou majeure), `exploratory` (angle jamais considéré,
  à ouvrir).
```

- [ ] **Step 3: Créer `scripts/devil-brain-schema.json`** (contenu complet)

```json
{
  "type": "object",
  "required": ["assessment", "questions"],
  "properties": {
    "assessment": {
      "type": "string",
      "description": "Impression generale en une ligne (contexte non evaluatif, pas un verdict)"
    },
    "questions": {
      "type": "array",
      "maxItems": 5,
      "items": {
        "type": "object",
        "required": ["question", "domain", "risk", "criticality"],
        "properties": {
          "question": {
            "type": "string",
            "description": "Question formulee pour etre posee telle quelle au porteur du projet"
          },
          "domain": {
            "type": "string",
            "description": "Domaine concerne (formulation libre)"
          },
          "risk": {
            "type": "string",
            "description": "Ce qui se passe concretement si on ecrit la spec sans la reponse"
          },
          "criticality": {
            "type": "string",
            "enum": ["blocking", "important", "exploratory"],
            "description": "blocking: specifier sans repondre serait dangereux; important: zone de flou majeure; exploratory: angle jamais considere"
          }
        }
      }
    }
  }
}
```

- [ ] **Step 4: Vérifier**

```bash
command jq . scripts/devil-spec-schema.json >/dev/null && echo SPEC_OK
command jq . scripts/devil-brain-schema.json >/dev/null && echo BRAIN_OK
ls scripts/
```
Attendu : `SPEC_OK`, `BRAIN_OK`, et `scripts/` contient exactement `devil-spec-mission.md`, `devil-spec-schema.json`, `devil-brain-mission.md`, `devil-brain-schema.json`.

- [ ] **Step 5: Commit**

```bash
command git add scripts/
command git commit -m "feat(contracts): missions et schémas par exercice (spec renommé, brain créé)"
```

---

### Task 2: Agent devil-gemini généralisé (transport pur)

**Files:**
- Rename+rewrite: `agents/devil-spec-gemini.md` → `agents/devil-gemini.md`

**Interfaces:**
- Consumes: le « Contrat de spawn agent » du header (MISSION_FILE, SCHEMA_FILE, VALIDATE_JQ, INPUTS).
- Produces: agent `devil-gemini` retournant l'enveloppe du header. Le sémantique v0.1.0 absorbée : l'instruction « lis-les intégralement » migre de l'ancien Step 1 vers le prompt assemblé générique (le diff v0.1.0 mission/agents du 2026-07-18 a confirmé qu'il n'y a RIEN d'autre à fusionner dans la mission).

- [ ] **Step 1: Renommer**

```bash
command git mv agents/devil-spec-gemini.md agents/devil-gemini.md
```

- [ ] **Step 2: Réécrire `agents/devil-gemini.md`** (contenu complet)

````markdown
---
name: devil-gemini
description: Transport Gemini des avocats du diable — assemble mission + inputs étiquetés, appelle Antigravity CLI (agy), retourne l'enveloppe JSON. L'exercice est porté par la mission fournie.
color: red
tools: Bash, Read, Glob, Grep
model: sonnet
---

Tu es le wrapper de transport du devil Gemini. Tu ne connais pas l'exercice :
la mission fournie le porte. Tu assembles, tu appelles, tu parses, tu
enveloppes.

## Entrée (fournie dans ton prompt)

```
MISSION_FILE=…            # chemin absolu
SCHEMA_FILE=…             # chemin absolu
VALIDATE_JQ=…             # expression jq -e de conformité minimale (une ligne)
INPUTS:                   # 1..N lignes LABEL:CHEMIN_ABSOLU
BRAINSTORMING:…
SPECS:…                   # exemple à 2 inputs ; il peut n'y en avoir qu'un
```

Avant le bash, transcris chaque ligne d'INPUTS en variables numérotées, dans
l'ordre reçu : `IN1_LABEL`/`IN1_PATH`, `IN2_LABEL`/`IN2_PATH`, … Pose
VALIDATE_JQ en bash entre single quotes.

## Sortie (contrat strict)

Ton message final est UN objet JSON sur une ligne, rien d'autre :
- succès : `{"devil":"gemini","model":"Gemini 3.5 Flash (High)","status":"ok","review":{…}}`
- échec  : `{"devil":"gemini","model":"Gemini 3.5 Flash (High)","status":"error","error":"CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT","detail":"≤ 500 chars"}`

## Procédure

IMPORTANT — passe un `timeout` explicite de **540000** ms (9 min) à ton appel
Bash qui lance agy : le défaut de 2 min couperait le run. Côté agy,
`--print-timeout 8m` fait le plafond en dessous. Jamais de rm : `trash`.

Contrat d'invocation agy (appris du plugin antigravity-plugin-cc) :
- `--print` est le DERNIER flag avant le prompt (le parseur Go consomme le token suivant).
- `< /dev/null` après le prompt est obligatoire : sans lui agy peut bloquer hors TTY.
- La sortie est ÉCRITE dans un fichier par agy (write_file), jamais lue depuis
  stdout : le stdout de `agy --print` peut revenir vide selon les versions
  (bug amont #76) alors que le modèle a répondu.

### Step 1 — Résoudre et assembler le prompt

Exemple à 2 inputs — adapte le nombre de lignes `IN*` à tes INPUTS :

```bash
IN1_ABS=$(realpath "$IN1_PATH"); IN2_ABS=$(realpath "$IN2_PATH")
SCHEMA=$(tr -d '\n' < "${SCHEMA_FILE}")
TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/devil-gemini-XXXXXX")
OUT_FILE="$TMP_DIR/review.json"
PROMPT_FILE="$TMP_DIR/prompt.txt"
{
  cat "${MISSION_FILE}"
  printf '\nTu as accès aux fichiers suivants. Lis-les intégralement avant de travailler :\n'
  printf -- '- %s : %s\n' "$IN1_LABEL" "$IN1_ABS"
  printf -- '- %s : %s\n' "$IN2_LABEL" "$IN2_ABS"
  printf '\nSortie STRICTEMENT conforme à ce schéma JSON : %s\n' "$SCHEMA"
  printf 'OUTPUT (CRITIQUE) : ne mets PAS ta sortie dans le chat. Écris le seul objet JSON brut via le tool write_file dans : %s\nAprès écriture, confirme uniquement le chemin. C est ton seul livrable.\n' "$OUT_FILE"
} > "$PROMPT_FILE"
```

### Step 2 — Appeler agy (timeout Bash 540000)

`--add-dir` pour le dossier de CHAQUE input, plus TMP_DIR :

```bash
agy --dangerously-skip-permissions \
  --add-dir "$(dirname "$IN1_ABS")" --add-dir "$(dirname "$IN2_ABS")" --add-dir "$TMP_DIR" \
  --model 'Gemini 3.5 Flash (High)' --print-timeout 8m \
  --print "$(cat "$PROMPT_FILE")" < /dev/null
```

### Step 3 — Parser et valider

```bash
REVIEW=$(command jq -c . "$OUT_FILE" 2>/dev/null)
printf '%s' "$REVIEW" | command jq -e "$VALIDATE_JQ" >/dev/null 2>&1 && VALID=yes || VALID=no
```

Si `REVIEW` est vide ou `VALID=no` : fais UN retry complet (Step 2 puis
Step 3, même OUT_FILE). Toujours en échec après retry :
- `OUT_FILE` absent ou vide → error `CLI_FAILED` (detail : 500 premiers chars
  du stdout agy capturé)
- fichier présent mais pas un JSON (`jq -c` échoue) → error `PARSE_ERROR`
  (detail : 500 premiers chars de OUT_FILE)
- JSON valide mais `VALID=no` → error `SCHEMA_INVALID` (detail : rédige
  TOI-MÊME les écarts constatés entre ce JSON et le schéma — champs
  manquants, enum inconnue, plus de 5 questions… — puis un court extrait du
  JSON ; ≤ 500 chars au total)
- appel Bash tué par timeout → error `TIMEOUT`

### Step 4 — Enveloppe et nettoyage

```bash
command jq -n -c --argjson review "$REVIEW" '{devil:"gemini",model:"Gemini 3.5 Flash (High)",status:"ok",review:$review}'
trash "$TMP_DIR" 2>/dev/null || true
```

En cas d'échec, construis l'enveloppe error avec `jq -n -c --arg detail "…"`.
Retourne l'enveloppe telle quelle. Ne formate pas, ne commente pas, n'ajoute
AUCUN texte autour.
````

- [ ] **Step 3: Vérifier le frontmatter et l'absence de sémantique spec**

```bash
command grep -n 'name: devil-gemini' agents/devil-gemini.md
command grep -c 'BRAINSTORM_FILE\|SPECS_FILE\|devil-spec' agents/devil-gemini.md
```
Attendu : ligne 2 matchée ; second grep = `0` (exit 1, normal).

- [ ] **Step 4: Commit**

```bash
command git add agents/
command git commit -m "feat(agents): devil-gemini généralisé en transport pur"
```

---

### Task 3: Agents devil-glm généralisé + devil-deepseek régénéré

**Files:**
- Rename+rewrite: `agents/devil-spec-glm.md` → `agents/devil-glm.md`
- Delete+regenerate: `agents/devil-spec-deepseek.md` → `agents/devil-deepseek.md` (par sed depuis devil-glm.md)

**Interfaces:**
- Consumes: « Contrat de spawn agent » du header.
- Produces: agents `devil-glm` et `devil-deepseek`, mêmes enveloppes que Task 2 (devil/model adaptés).

- [ ] **Step 1: Renommer glm, supprimer l'ancien deepseek (régénéré ensuite)**

```bash
command git mv agents/devil-spec-glm.md agents/devil-glm.md
command git rm agents/devil-spec-deepseek.md
```

(`git rm` retire du versionnement — c'est le remplacement planifié par régénération, pas une perte : le fichier reste dans l'historique git.)

- [ ] **Step 2: Réécrire `agents/devil-glm.md`** (contenu complet)

````markdown
---
name: devil-glm
description: Transport GLM des avocats du diable — assemble mission + inputs étiquetés, appelle claude CLI sur ollama cloud (glm-5.2:cloud), retourne l'enveloppe JSON. L'exercice est porté par la mission fournie.
color: red
tools: Bash, Read, Glob, Grep
model: sonnet
---

Tu es le wrapper de transport du devil GLM. Tu ne connais pas l'exercice :
la mission fournie le porte. Tu assembles, tu appelles, tu parses, tu
enveloppes.

## Entrée (fournie dans ton prompt)

```
MISSION_FILE=…            # chemin absolu
SCHEMA_FILE=…             # chemin absolu
VALIDATE_JQ=…             # expression jq -e de conformité minimale (une ligne)
INPUTS:                   # 1..N lignes LABEL:CHEMIN_ABSOLU
BRAINSTORMING:…
SPECS:…                   # exemple à 2 inputs ; il peut n'y en avoir qu'un
```

Avant le bash, transcris chaque ligne d'INPUTS en variables numérotées, dans
l'ordre reçu : `IN1_LABEL`/`IN1_PATH`, `IN2_LABEL`/`IN2_PATH`, … Pose
VALIDATE_JQ en bash entre single quotes.

## Sortie (contrat strict)

Ton message final est UN objet JSON sur une ligne, rien d'autre :
- succès : `{"devil":"glm","model":"glm-5.2:cloud","status":"ok","review":{…}}`
- échec  : `{"devil":"glm","model":"glm-5.2:cloud","status":"error","error":"CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT","detail":"≤ 500 chars"}`

## Procédure

IMPORTANT — passe un `timeout` explicite de **540000** ms (9 min) à chaque
appel Bash qui lance le modèle : le défaut de 2 min couperait le run.
Jamais de `ollama pull`, jamais de préflight de présence du modèle (les
modèles `:cloud` ne sont pas listés par `/api/tags`, c'est normal et validé).
Jamais de rm : `trash`.

### Step 1 — Résoudre et assembler le prompt

Le run modèle est hermétique (aucun outil) : les CONTENUS des inputs sont
embarqués, chacun entre marqueurs portant son label. Exemple à 2 inputs —
adapte le nombre de blocs à tes INPUTS :

```bash
IN1_ABS=$(realpath "$IN1_PATH"); IN2_ABS=$(realpath "$IN2_PATH")
SCHEMA=$(tr -d '\n' < "${SCHEMA_FILE}")
TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/devil-glm-XXXXXX")
PROMPT_FILE="$TMP_DIR/prompt.txt"
{
  cat "${MISSION_FILE}"
  printf '\n=== BEGIN %s ===\n' "$IN1_LABEL"
  cat "$IN1_ABS"
  printf '\n=== END %s ===\n' "$IN1_LABEL"
  printf '\n=== BEGIN %s ===\n' "$IN2_LABEL"
  cat "$IN2_ABS"
  printf '\n=== END %s ===\n' "$IN2_LABEL"
  printf '\nSortie STRICTEMENT conforme à ce schéma JSON : %s\n' "$SCHEMA"
  printf 'OUTPUT : imprime UNIQUEMENT le seul objet JSON brut. Pas de fence markdown, pas de texte avant ou après.\n'
} > "$PROMPT_FILE"
```

### Step 2 — Appeler GLM (timeout Bash 540000)

Ligne de base validée par Romain + flags d'hermétisme validés le 2026-07-18 :

```bash
RAW=$(cd "$TMP_DIR" && ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434 \
  ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max \
  claude --model glm-5.2:cloud --dangerously-skip-permissions \
  --strict-mcp-config --tools "" --setting-sources "" --no-session-persistence \
  -p --output-format json < "$PROMPT_FILE" 2>"$TMP_DIR/stderr.log")
```

### Step 3 — Parser et valider

```bash
RESULT=$(printf '%s' "$RAW" | command jq -r 'select(.type=="result") | .result // empty' 2>/dev/null)
IS_ERR=$(printf '%s' "$RAW" | command jq -r '.is_error' 2>/dev/null)
REVIEW=$(printf '%s' "$RESULT" | sed -e '/^```/d' | command jq -c '.' 2>/dev/null)
printf '%s' "$REVIEW" | command jq -e "$VALIDATE_JQ" >/dev/null 2>&1 && VALID=yes || VALID=no
```

Si `RAW` est vide, ou `IS_ERR` vaut `true`, ou `VALID=no` : fais UN retry
complet (Step 2 puis Step 3). Toujours en échec après retry, choisis le
detail LE PLUS DIAGNOSTIQUE — surtout PAS le warning « connectors are
disabled » de stderr, qui est du bruit constant et trompeur :

```bash
API_STATUS=$(printf '%s' "$RAW" | command jq -r '.api_error_status // empty' 2>/dev/null)
ERR_MSG=$(printf '%s' "$RAW" | command jq -r '.result // empty' 2>/dev/null)
# priorité : message d'erreur amont (souvent présent même si is_error=true) > stderr
DETAIL=$(printf '%s' "${API_STATUS:+[$API_STATUS] }${ERR_MSG:-$(head -c 500 "$TMP_DIR/stderr.log")}" | head -c 500)
```

- `RAW` vide ou `IS_ERR=true` → error `CLI_FAILED` (detail = `$DETAIL` ci-dessus)
- `RESULT` non-JSON (`jq -c` échoue) → error `PARSE_ERROR` (detail : 500
  premiers chars de RESULT, sinon de RAW)
- JSON valide mais `VALID=no` → error `SCHEMA_INVALID` (detail : rédige
  TOI-MÊME les écarts constatés entre ce JSON et le schéma — champs
  manquants, enum inconnue, plus de 5 questions… — puis un court extrait du
  JSON ; ≤ 500 chars au total)
- appel Bash tué par timeout → error `TIMEOUT`

### Step 4 — Enveloppe et nettoyage

```bash
command jq -n -c --argjson review "$REVIEW" '{devil:"glm",model:"glm-5.2:cloud",status:"ok",review:$review}'
trash "$TMP_DIR" 2>/dev/null || true
```

En cas d'échec, construis l'enveloppe error avec `jq -n -c --arg detail "…"`.
Retourne l'enveloppe telle quelle. Ne formate pas, ne commente pas, n'ajoute
AUCUN texte autour.
````

- [ ] **Step 3: Régénérer devil-deepseek par sed (MODÈLE d'abord) + contrôle post-génération**

```bash
command sed -e 's/glm-5\.2:cloud/deepseek-v4-pro:cloud/g' -e 's/glm/deepseek/g' -e 's/GLM/Deepseek/g' agents/devil-glm.md > agents/devil-deepseek.md
command grep -ci 'glm' agents/devil-deepseek.md
command grep -c 'deepseek-v4-pro:cloud' agents/devil-deepseek.md
command grep -n 'name: devil-deepseek' agents/devil-deepseek.md
```
Attendu : premier grep `0` (exit 1, insensible à la casse — aucun résidu glm/GLM) ; deuxième ≥ `3` ; troisième matche la ligne 2.

- [ ] **Step 4: Vérifier l'absence de sémantique spec dans les deux fichiers**

```bash
command grep -c 'BRAINSTORM_FILE\|SPECS_FILE\|devil-spec' agents/devil-glm.md agents/devil-deepseek.md
```
Attendu : `0` pour chacun (exit 1).

- [ ] **Step 5: Commit**

```bash
command git add agents/
command git commit -m "feat(agents): devil-glm généralisé + devil-deepseek régénéré par sed"
```

---

### Task 4: Migration des skills devil-spec et devil-spec-swarm

**Files:**
- Modify: `skills/devil-spec/SKILL.md` (étapes 0 et 2)
- Modify: `skills/devil-spec-swarm/SKILL.md` (étapes 0 et 2)

**Interfaces:**
- Consumes: agents Tasks 2-3 (`devil:devil-{gemini,glm,deepseek}`), contrats Task 1, « Contrat de spawn agent » + VALIDATE_JQ spec du header.
- Produces: surfaces `/devil-spec` et `/devil-spec-swarm` inchangées pour l'utilisateur.

- [ ] **Step 1: `skills/devil-spec/SKILL.md` — étape 0, remplacer les deux lignes de chemins**

Ancien :
```
  - `SCHEMA_FILE` = `<racine>/scripts/spec-review-schema.json`
  - `MISSION_FILE` = `<racine>/scripts/devil-mission.md`
```
Nouveau :
```
  - `SCHEMA_FILE` = `<racine>/scripts/devil-spec-schema.json`
  - `MISSION_FILE` = `<racine>/scripts/devil-spec-mission.md`
```

- [ ] **Step 2: `skills/devil-spec/SKILL.md` — étape 2, remplacer le bloc de spawn**

Ancien (bloc complet, du titre « ## Étape 2 — Lancer le sous-agent » jusqu'à la ligne d'annonce incluse) :
```
Spawn l'agent du devil choisi — `devil:devil-spec-gemini`,
`devil:devil-spec-glm` ou `devil:devil-spec-deepseek` (si le nom préfixé
n'est pas résolu par le harness, réessaie sans préfixe,
ex. `devil-spec-glm`) :

```
Agent(
  subagent_type: "devil:devil-spec-<devil>",
  prompt: "BRAINSTORM_FILE=<abs>\nSPECS_FILE=<abs>\nSCHEMA_FILE=<abs>\nMISSION_FILE=<abs>\n\nExécute la procédure de review."
)
```
```
Nouveau :
```
Spawn l'agent du devil choisi — `devil:devil-gemini`, `devil:devil-glm` ou
`devil:devil-deepseek` ; si ce type est introuvable (plugin non chargé),
retente sans préfixe, ex. `devil-glm` :

```
Agent(
  subagent_type: "devil:devil-<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.verdict | IN(\"approve\",\"rework\",\"reject\"))\nINPUTS:\nBRAINSTORMING:<abs brainstorm>\nSPECS:<abs specs>\n\nExécute la procédure de transport."
)
```
```
(la ligne d'annonce « Review en cours… » reste inchangée sous le bloc)

- [ ] **Step 3: `skills/devil-spec-swarm/SKILL.md` — étape 0, mêmes deux chemins que Step 1**

Remplacer `spec-review-schema.json` → `devil-spec-schema.json` et `devil-mission.md` → `devil-spec-mission.md` dans la phrase de l'étape 0.

- [ ] **Step 4: `skills/devil-spec-swarm/SKILL.md` — étape 2, types + prompt**

Remplacer la phrase des types :
```
subagent_type change : `devil:devil-spec-gemini`, `devil:devil-spec-glm`,
`devil:devil-spec-deepseek` (fallback sans préfixe si non résolu) :
```
par :
```
subagent_type change : `devil:devil-gemini`, `devil:devil-glm`,
`devil:devil-deepseek` (si le type est introuvable, fallback sans préfixe) :
```
Et remplacer le bloc prompt :
```
prompt: "BRAINSTORM_FILE=<abs>\nSPECS_FILE=<abs>\nSCHEMA_FILE=<abs>\nMISSION_FILE=<abs>\n\nExécute la procédure de review."
```
par le MÊME prompt que Step 2 de cette task (MISSION_FILE, SCHEMA_FILE, VALIDATE_JQ spec, INPUTS BRAINSTORMING + SPECS, « Exécute la procédure de transport. »).

- [ ] **Step 5: Vérifier zéro référence obsolète dans les deux skills**

```bash
command grep -n 'devil-spec-gemini\|devil-spec-glm\|devil-spec-deepseek\|devil-mission\.md\|spec-review-schema\.json\|BRAINSTORM_FILE\|SPECS_FILE=' skills/devil-spec/SKILL.md skills/devil-spec-swarm/SKILL.md
```
Attendu : aucune sortie (exit 1).

- [ ] **Step 6: Commit**

```bash
command git add skills/devil-spec/ skills/devil-spec-swarm/
command git commit -m "refactor(skills): devil-spec et devil-spec-swarm sur les agents transport généralisés"
```

---

### Task 5: Skill /devil-brain (unitaire)

**Files:**
- Create: `skills/devil-brain/SKILL.md`

**Interfaces:**
- Consumes: agents Tasks 2-3, contrats brain Task 1, « Contrat de spawn agent » + VALIDATE_JQ brain du header.
- Produces: surface `/devil-brain [devil] [fichier]`.

- [ ] **Step 1: Créer `skills/devil-brain/SKILL.md`** (contenu complet)

````markdown
---
name: devil-brain
description: "Interrogatoire socratique d'un doc de brainstorming par un avocat du diable au choix (Gemini par défaut, GLM, Deepseek) : les 5 questions les plus dangereuses jamais posées, sans score ni verdict. Triggers: /devil-brain, 'questionne le brainstorm', 'angles morts du brainstorming'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-brain — L'interrogatoire socratique du brainstorming

Un devil externe lit un doc de brainstorming (définition de besoins) et rend
les questions les PLUS DANGEREUSES jamais posées — 5 max, sans score ni
verdict. Toi (l'orchestrateur) tu restitues, Romain écarte ou retient, et tu
amendes le doc au fil de ses réponses.

## Syntaxe

```
/devil-brain                          # auto-detect, devil gemini
/devil-brain glm                      # auto-detect, devil glm
/devil-brain chemin/brainstorming.md  # fichier explicite, gemini
/devil-brain chemin/brainstorming.md deepseek
```

## Étape 0 — Résoudre le devil et les chemins du plugin

- Devil : l'argument s'il vaut `gemini`, `glm` ou `deepseek` ; sinon `gemini`.
- Racine du plugin : deux niveaux au-dessus du « Base directory for this
  skill » injecté ci-dessus. Résous en absolu :
  - `SCHEMA_FILE` = `<racine>/scripts/devil-brain-schema.json`
  - `MISSION_FILE` = `<racine>/scripts/devil-brain-mission.md`
- Vérifie que les deux existent (Read). S'ils manquent, arrête et signale un
  plugin corrompu.

## Étape 1 — Identifier le doc de brainstorming

- Path fourni en argument → vérifie qu'il existe et n'est pas vide.
  Introuvable ou vide → arrête et demande, AUCUN appel modèle.
- Sinon auto-detect : Glob `**/brainstorming.md` dans `.specs/` ; plusieurs
  chantiers → demande lequel ; un seul → prends-le.
- En pleine session de brainstorming sans fichier : écris d'abord l'état
  courant de la compréhension dans `.specs/<chantier>/brainstorming.md`,
  puis continue avec ce fichier.

Confirmation avant lancement :

> **Interrogatoire :** <devil> questionne `<fichier>` (jusqu'à 9 min). Go ?

## Étape 2 — Lancer le sous-agent

Spawn `devil:devil-<devil>` ; si ce type est introuvable (plugin non
chargé), retente `devil-<devil>` sans préfixe :

```
Agent(
  subagent_type: "devil:devil-<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"assessment\") and has(\"questions\") and (.questions|type==\"array\" and length<=5) and (.questions|all(has(\"question\") and has(\"domain\") and has(\"risk\") and (.criticality|IN(\"blocking\",\"important\",\"exploratory\"))))\nINPUTS:\nBRAINSTORMING:<abs>\n\nExécute la procédure de transport."
)
```

Annonce avant le spawn : « **Interrogatoire en cours…** <devil> cherche les
questions jamais posées (jusqu'à 9 min). »

## Étape 3 — Parser l'enveloppe et restituer

Le retour de l'agent est UNE ligne JSON : `{devil, model, status,
review|error+detail}`.

### Si `status: "error"`
```
══════ INTERROGATOIRE ÉCHOUÉ ══════

Devil : <devil> (<model>)
Erreur : <error> — <detail>

→ Relance (/devil-brain <devil>), autre devil (/devil-brain glm|deepseek|gemini),
  ou les trois voix (/devil-brain-swarm).
```

### Si `status: "ok"` et `questions` vide
> **Rien de dangereux à signaler.** <assessment>
> Le devil n'a pas de question à poser : tu peux passer aux specs.

(Pas de Q&A forcé. Fin de la skill.)

### Si `status: "ok"` avec questions
```
══════ DEVIL BRAIN ══════

Devil : <devil> (<model>)
Assessment : <assessment>

| # | Criticité | Domaine | Question | Risque si on spécifie sans répondre |
```
Tri : blocking > important > exploratory. Criticités affichées en français :
BLOQUANTE / IMPORTANTE / EXPLORATOIRE.

## Étape 4 — Tri puis Q&A ciblé

1. Demande à Romain lesquelles il ÉCARTE (réponse d'un mot : « 2 et 5 »,
   « aucune », « toutes »…).
2. Questions écartées : propose de les tracer en non-buts explicites du doc
   de brainstorming. Romain tranche ; si oui, ajoute chaque sujet écarté à
   la section Non-buts via Edit.
3. Questions retenues : pose-les UNE PAR UNE. Après chaque réponse de
   Romain, amende le doc de brainstorming via Edit — la décision s'intègre
   dans la section qu'elle concerne (ou une nouvelle section), jamais en
   annexe fourre-tout.
4. Fin : récap une ligne par question — intégrée (où) ou écartée (non-but ou
   simple abandon).

## Règles

- Tu ne modifies QUE le doc de brainstorming, et uniquement avec les
  réponses de Romain — jamais tes propres réponses aux questions.
- Pas de re-passage automatique après amendement : le re-appel est toujours
  manuel (/devil-brain ou /devil-brain-swarm).
- Le devil ne modifie rien : c'est toi qui amendes.
````

- [ ] **Step 2: Vérifier**

```bash
command grep -n 'name: devil-brain' skills/devil-brain/SKILL.md
command grep -c '~/.claude' skills/devil-brain/SKILL.md
```
Attendu : ligne 2 matchée ; second grep `0` (exit 1).

- [ ] **Step 3: Commit**

```bash
command git add skills/devil-brain/
command git commit -m "feat(skills): /devil-brain — interrogatoire socratique unitaire"
```

---

### Task 6: Skill /devil-brain-swarm (trois voix)

**Files:**
- Create: `skills/devil-brain-swarm/SKILL.md`

**Interfaces:**
- Consumes: agents Tasks 2-3, contrats brain Task 1, prompt de spawn identique à Task 5 étape 2.
- Produces: surface `/devil-brain-swarm [fichier]`.

- [ ] **Step 1: Créer `skills/devil-brain-swarm/SKILL.md`** (contenu complet)

````markdown
---
name: devil-brain-swarm
description: "Interrogatoire socratique à trois voix : Gemini + GLM + Deepseek questionnent le même doc de brainstorming en parallèle, questions consolidées par convergence (3/3, 2/3, 1/3), sans score ni verdict. Triggers: /devil-brain-swarm, 'swarm socratique', 'les devils sur le brainstorm'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-brain-swarm — L'interrogatoire à trois voix

Les 3 devils lisent le MÊME doc de brainstorming EN PARALLÈLE et rendent
chacun leurs 5 questions les plus dangereuses. Toi (l'orchestrateur) tu
consolides — équivalences, convergence, tri — puis tri + Q&A avec Romain.

## Étape 0 — Chemins du plugin

Identique à /devil-brain : racine = deux niveaux au-dessus du base directory
injecté ; résous `SCHEMA_FILE` = `<racine>/scripts/devil-brain-schema.json`
et `MISSION_FILE` = `<racine>/scripts/devil-brain-mission.md`, vérifie
l'existence.

## Étape 1 — Doc de brainstorming

Identique à /devil-brain (path explicite sinon auto-detect
`**/brainstorming.md` dans `.specs/` ; introuvable ou vide → stop sans
appel ; en pleine session sans fichier → écris d'abord le draft).
Confirmation :

> **Interrogatoire à trois voix :** gemini + glm + deepseek questionnent
> `<fichier>` en parallèle (jusqu'à 9 min). Go ?

## Étape 2 — Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message (c'est ce qui
les fait tourner en parallèle). Même prompt que /devil-brain étape 2 (même
MISSION_FILE, SCHEMA_FILE, VALIDATE_JQ, INPUTS `BRAINSTORMING:<abs>`), seul
le subagent_type change : `devil:devil-gemini`, `devil:devil-glm`,
`devil:devil-deepseek` (si un type est introuvable, fallback sans préfixe).

## Étape 3 — Collecte et quorum

Chaque retour est une enveloppe `{devil, model, status, review|error}`.

- 3 voix `ok` → consolidation pleine.
- 2 voix `ok` → continue, et le rapport OUVRE sur la voix absente :
  « ⚠ <devil> muet (<error> — <detail court>). Consolidation sur 2 voix. »
- ≤ 1 voix `ok` → PAS de consolidation. Rapport d'échec avec le détail des
  erreurs, proposer : relancer le swarm, ou l'unitaire (/devil-brain <devil>).

Un retour qui n'est pas une enveloppe JSON valide compte comme voix absente
(ne JAMAIS interpréter un texte d'erreur comme des questions).

## Étape 4 — Consolidation (ton travail d'orchestrateur)

Opérée par TOI, capacité LLM native — aucun algorithme, embedding ni seuil :

- Deux questions sont ÉQUIVALENTES si elles visent le même angle mort ou
  nomment le même risque, même formulées différemment.
- Chaque groupe : badge de convergence 3/3, 2/3, 1/3 (sur les voix
  exprimées : à 2 voix, 2/2 et 1/2) ; criticité du groupe = la plus haute
  parmi les voix groupées ; garde la formulation la plus incisive et la
  plus spécifique au projet.
- Les singletons restent au tableau avec leur badge 1/3.
- TRI : criticité d'abord (blocking > important > exploratory), convergence
  ensuite. Une bloquante 1/3 passe devant une exploratoire 3/3.

## Étape 5 — Rapport

```
══════ INTERROGATOIRE À TROIS VOIX ══════

[⚠ voix absente le cas échéant]

Assessments : gemini « … » · glm « … » · deepseek « … »

| # | Criticité | Conv | Domaine | Question | Risque | Devils |
```
Criticités affichées en français : BLOQUANTE / IMPORTANTE / EXPLORATOIRE.

Si TOUTES les voix exprimées rendent 0 question :
> **Rien à signaler : aucune voix n'a de question dangereuse.**
> Assessments : <une ligne par devil>. Tu peux passer aux specs.

(Pas de Q&A forcé. Fin de la skill.)

## Étape 6 — Tri puis Q&A ciblé

Identique à /devil-brain étape 4 : Romain écarte d'un regard ; écartées →
non-buts proposés dans le doc ; retenues posées UNE PAR UNE ; doc de
brainstorming amendé via Edit au fil des réponses ; récap final une ligne
par question.

## Règles

- Tu ne modifies QUE le doc de brainstorming, et uniquement avec les
  réponses de Romain.
- Pas de re-passage automatique après amendement : re-appel manuel.
- Coût : 3 modèles en parallèle ≈ la durée du plus lent. C'est un gate de
  définition de besoins, pas un lint : pas sur un brouillon de trois lignes.
````

- [ ] **Step 2: Vérifier**

```bash
command grep -n 'name: devil-brain-swarm' skills/devil-brain-swarm/SKILL.md
command grep -c '~/.claude' skills/devil-brain-swarm/SKILL.md
```
Attendu : ligne 2 matchée ; second grep `0` (exit 1).

- [ ] **Step 3: Commit**

```bash
command git add skills/devil-brain-swarm/
command git commit -m "feat(skills): /devil-brain-swarm — trois voix consolidées par convergence"
```

---

### Task 7: Manifest v0.2.0, README, inventaire propre

**Files:**
- Modify: `.claude-plugin/plugin.json` (version, description, keywords)
- Modify: `README.md` (usage + entrées)

**Interfaces:**
- Consumes: tout ce qui précède.
- Produces: plugin v0.2.0 auto-cohérent, prêt pour les smokes.

- [ ] **Step 1: `.claude-plugin/plugin.json` — contenu complet**

```json
{
  "$schema": "https://www.schemastore.org/claude-code-plugin-manifest.json",
  "name": "devil",
  "description": "Avocats du diable : reviews critiques de specs (score, verdict, issues) et interrogatoire socratique de brainstormings (les questions jamais posées), par Gemini, GLM et Deepseek, unitaires ou en swarm.",
  "version": "0.2.0",
  "author": {
    "name": "Romain Ecarnot",
    "url": "https://github.com/eRom"
  },
  "repository": "https://github.com/eRom/erom-agence-devil",
  "license": "MIT",
  "keywords": ["review", "specs", "brainstorming", "devils-advocate", "socratic", "gemini", "glm", "deepseek", "swarm"],
  "skills": "./skills/"
}
```
(Toujours PAS de clé `agents` : rejetée par le schéma, auto-découverte.)

- [ ] **Step 2: `README.md` — remplacer le bloc Usage et le paragraphe suivant**

Ancien :
```
```
/devil-spec                          # auto-detect dans .specs/, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
```

Entrées : toujours 2 fichiers (brainstorming + specs). Sortie par devil :
JSON strict (score 0-100, verdict approve/rework/reject, 6 critères, issues
actionnables). Le swarm consolide : VALABLE, MODIFICATIONS REQUISES, ou
JETABLE, avec convergence des issues (3/3, 2/3, 1/3) et voix dissonantes.
```
Nouveau :
```
```
/devil-spec                          # specs vs brainstorm, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
/devil-brain                         # questions socratiques sur un brainstorming
/devil-brain brainstorming.md deepseek
/devil-brain-swarm                   # les 3 voix, consolidation par convergence
```

**devil-spec** — entrées : 2 fichiers (brainstorming + specs). Sortie par
devil : JSON strict (score 0-100, verdict approve/rework/reject, 6 critères,
issues actionnables). Le swarm consolide : VALABLE, MODIFICATIONS REQUISES,
ou JETABLE, avec convergence des issues (3/3, 2/3, 1/3) et voix dissonantes.

**devil-brain** — entrée : 1 fichier (brainstorming). Sortie par devil : les
5 questions les plus dangereuses jamais posées (domaine, risque, criticité)
plus une impression en une ligne — sans score ni verdict, 0 question = prêt
à spécifier. Le swarm consolide par convergence, tri criticité puis
convergence, puis Q&A ciblé qui amende le doc.
```

- [ ] **Step 3: Inventaire plugin (les agents et skills sont découverts)**

```bash
claude --plugin-dir /Users/recarnot/dev/erom-agence-devil plugin details devil
```
Attendu : version 0.2.0, 4 skills (devil-spec, devil-spec-swarm, devil-brain, devil-brain-swarm), 3 agents (devil-gemini, devil-glm, devil-deepseek).

- [ ] **Step 4: Grep final anti-références obsolètes (périmètre livré uniquement — `.specs/` et `_memory_/` sont de l'historique, exclus)**

```bash
command grep -rn 'devil-spec-gemini\|devil-spec-glm\|devil-spec-deepseek\|devil-mission\.md\|spec-review-schema\.json' skills/ agents/ scripts/ README.md .claude-plugin/
command grep -rn '~/.claude' skills/ agents/ scripts/
```
Attendu : aucune sortie pour les deux (exit 1).

- [ ] **Step 5: Commit**

```bash
command git add .claude-plugin/plugin.json README.md
command git commit -m "chore(plugin): v0.2.0 — manifest, README, quatre surfaces documentées"
```

---

### Task 8: Smokes live (erreur, brain, non-régression spec)

Aucun fichier créé (vérifications). Chaque spawn d'agent de test suit la leçon du 2026-07-18 : le prompt exige l'écriture de l'enveloppe dans un fichier scratchpad EN PLUS du retour (le canal de retour des teammates est intermittent).

- [ ] **Step 1: Chemin d'erreur (script direct, modèle bidon — sans agent, déterministe)**

```bash
set +e
TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/devil-err-XXXXXX")
printf 'Réponds {"ok":true} en JSON brut.\n' > "$TMP_DIR/prompt.txt"
RAW=$(cd "$TMP_DIR" && ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434 \
  ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max \
  claude --model glm-BIDON:cloud --dangerously-skip-permissions \
  --strict-mcp-config --tools "" --setting-sources "" --no-session-persistence \
  -p --output-format json < "$TMP_DIR/prompt.txt" 2>"$TMP_DIR/stderr.log")
IS_ERR=$(printf '%s' "$RAW" | command jq -r '.is_error' 2>/dev/null)
API_STATUS=$(printf '%s' "$RAW" | command jq -r '.api_error_status // empty' 2>/dev/null)
ERR_MSG=$(printf '%s' "$RAW" | command jq -r '.result // empty' 2>/dev/null)
DETAIL=$(printf '%s' "${API_STATUS:+[$API_STATUS] }${ERR_MSG:-$(head -c 500 "$TMP_DIR/stderr.log")}" | head -c 500)
echo "IS_ERR=$IS_ERR" ; echo "DETAIL=$DETAIL"
trash "$TMP_DIR"
```
(timeout Bash : 540000). Attendu : `IS_ERR=true` et `DETAIL` commençant par `[404]` — le chemin diagnostique du transport tient toujours.

- [ ] **Step 2: Smoke brain unitaire (glm) sur la fixture**

Spawn UN agent `general-purpose` avec ce prompt (chemins absolus réels) :
```
Lis /Users/recarnot/dev/erom-agence-devil/agents/devil-glm.md et exécute
EXACTEMENT sa procédure (corps, pas le frontmatter) avec :
MISSION_FILE=/Users/recarnot/dev/erom-agence-devil/scripts/devil-brain-mission.md
SCHEMA_FILE=/Users/recarnot/dev/erom-agence-devil/scripts/devil-brain-schema.json
VALIDATE_JQ=has("assessment") and has("questions") and (.questions|type=="array" and length<=5) and (.questions|all(has("question") and has("domain") and has("risk") and (.criticality|IN("blocking","important","exploratory"))))
INPUTS:
BRAINSTORMING:/Users/recarnot/dev/erom-agence-devil/examples/brainstorming.md

Écris l'enveloppe finale dans <scratchpad>/smoke-brain-glm.json ET
retourne-la. Uniquement le JSON, aucun texte autour.
```
Attendu : enveloppe `status:"ok"`, 1 à 5 questions, chacune spécifique au projet veilleur (pas de récitation de grille), champs complets, criticality dans l'enum.

- [ ] **Step 3: Non-régression spec — les 3 agents généralisés sur les fixtures veilleur**

Spawn les 3 agents `general-purpose` EN UN SEUL message, prompts identiques au Step 2 sauf : fichier agent respectif (`devil-gemini.md`, `devil-glm.md`, `devil-deepseek.md`), MISSION_FILE=`scripts/devil-spec-mission.md`, SCHEMA_FILE=`scripts/devil-spec-schema.json`, VALIDATE_JQ spec (header), INPUTS :
```
BRAINSTORMING:/Users/recarnot/dev/erom-agence-devil/examples/brainstorming.md
SPECS:/Users/recarnot/dev/erom-agence-devil/examples/specs.md
```
et fichiers scratchpad `smoke-spec-<devil>.json`.
Attendu : 3 enveloppes `ok`, les 3 verdicts `reject` (référence v0.1.0 : gemini 15, glm 22, deepseek 12), les 6 défauts plantés retrouvés dans les issues consolidées (SaaS, OpenAI distant, clé en clair, SQLite vs JSON, export manquant, ni tests ni erreurs).

- [ ] **Step 4: Si un smoke échoue** — diagnostiquer avec l'enveloppe error (jamais stderr seul), corriger le fichier agent ou skill concerné, committer le fix (`fix(agents): …`), relancer le smoke concerné. Ne jamais passer à la Task 9 avec un smoke rouge.

---

### Task 9: Dogfood méta — /devil-brain-swarm sur son propre brainstorming

Vérification finale en conditions réelles, ET premier usage réel de la skill.

- [ ] **Step 1: Dérouler `skills/devil-brain-swarm/SKILL.md` à la main** (les types `devil:devil-*` ne seront chargés qu'après réinstallation — fallback prescrit par la skill : ici, spawns `general-purpose` comme en Task 8) sur :
```
BRAINSTORMING:/Users/recarnot/dev/erom-agence-devil/.specs/plugin-devil-brain/brainstorming.md
```

- [ ] **Step 2: Consolider selon l'étape 4 de la skill** (équivalence par angle mort/risque, criticité max par groupe, tri criticité puis convergence) et présenter le rapport de l'étape 5 à Romain.

- [ ] **Step 3: Q&A ciblé selon l'étape 6** — Romain écarte/retient ; amender `.specs/plugin-devil-brain/brainstorming.md` avec ses réponses ; commit final si le doc change (`docs(specs): réponses au dogfood devil-brain-swarm`).

---

### Task 10: Livraison (après merge sur main — hors branche)

PRÉALABLE : la branche est terminée via superpowers:finishing-a-development-branch (tests = Tasks 8-9 verts), Romain a choisi le merge local. NE PAS pousser sans son accord explicite.

- [ ] **Step 1: Marketplace** — dans `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json` : bump `metadata.version` `0.5.0` → `0.6.0` (l'entrée `devil` pointe l'URL GitHub, rien d'autre à changer). Commit + push (HTTPS déjà configuré).

- [ ] **Step 2: Push du repo devil** (origin HTTPS déjà configuré) :
```bash
command git push origin main
```

- [ ] **Step 3: Réinstallation (attendre ~10 s après les push)** :
```bash
claude plugin marketplace update erom-marketplace
claude plugin install devil@erom-marketplace
claude plugin details devil
```
Attendu : `devil@erom-marketplace v0.2.0`, scope user, enabled, 4 skills + 3 agents.

- [ ] **Step 4: Rappeler à Romain** que les nouvelles skills/agents ne seront visibles qu'après redémarrage des sessions Claude Code ouvertes.
