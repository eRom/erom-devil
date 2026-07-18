# Plan d'implémentation — Plugin « devil » v0.1.0

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Plugin Claude Code `devil` : 3 avocats du diable (Gemini, GLM, Deepseek) qui jugent une spec contre son brainstorm, unitairement (`/devil-spec`) ou en swarm avec verdict consolidé (`/devil-spec-swarm`).

**Architecture:** 3 agents wrappers Sonnet symétriques ; Gemini via agy (écriture fichier, bug stdout amont), GLM/Deepseek via `claude -p` sur ollama cloud en texte pur (stdin → stdout JSON). Contrat commun : schéma v2 à 3 verdicts + enveloppe `{devil, model, status, review|error}`. Skills : dispatch unitaire + swarm (3 spawns parallèles dans un seul message, synthèse codifiée).

**Tech Stack:** Plugin Claude Code (agents .md + skills), bash + jq + sed, agy CLI, claude CLI ≥ 2.1.214, ollama (endpoint Anthropic-compat local).

**Spec:** `.specs/plugin-devil/architecture-technique.md` (+ `brainstorming.md` amont).

## Global Constraints

- Jamais `rm`/`rmdir`/`unlink` : `trash` uniquement, partout, agents inclus.
- Ligne de base GLM/Deepseek de Romain reprise verbatim (env `ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434 ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max` + `--dangerously-skip-permissions`). JAMAIS de `ollama pull`, JAMAIS de préflight de présence du modèle (`/api/tags` ne liste pas les `:cloud`), aucun re-test de la connectivité de base (consigne explicite Romain).
- Flags d'hermétisme ajoutés, VALIDÉS live le 2026-07-18 sur gemma local : `--strict-mcp-config --tools "" --setting-sources "" --no-session-persistence`. stdout = JSON pur (le warning « connectors » part sur stderr) → toujours capturer stdout seul, stderr dans un fichier.
- Parsing enveloppe validé live : `.result` porte le texte, `.is_error`/`.subtype` pour les erreurs. Dry-run du pipeline sed+jq : OK.
- Timeout Bash explicite `540000` ms sur tout appel de modèle, 1 retry max.
- Aucun chemin `~/.claude/` en dur dans les fichiers du plugin.
- Textes user-facing en français ; identifiants techniques du schéma en anglais.
- Les devils ne modifient JAMAIS les fichiers d'entrée ; le brainstorm n'est JAMAIS modifié par les corrections.
- Commits fréquents (1 par task), trailers : `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` + `Claude-Session: https://claude.ai/code/session_013zG81zFmmGeLihXvXSxQ6u`.
- Hook rtk local : préfixer `command` tout appel git/grep/ls dont la sortie pilote une décision.

## Échantillons capturés (provenance)

| Convention | Source |
|---|---|
| Enveloppe `-p --output-format json` (`.result`, `.is_error`, `.subtype`) | Run live 2026-07-18, gemma local via ollama |
| Flags hermétisme acceptés + stdout pur | Idem, 2e run avec `2>/dev/null` |
| Pipeline parsing sed+jq | Dry-run local 2026-07-18 sur enveloppe capturée |
| `plugin.json` (champs, `$schema` schemastore, dossiers déclarés) | `~/.claude/plugins/cache/erom-marketplace/caserne/2.8.0/.claude-plugin/plugin.json` + superpowers 6.1.1 |
| Entrée marketplace | `erom-marketplace/.claude-plugin/marketplace.json` (entrée caserne, lue) |
| `subagent_type` plugin-qualifié `plugin:agent` | Registre agents de la session (ex. observé : `code-simplifier:code-simplifier`) |
| Contrat agy (`--print` dernier flag, `< /dev/null`, review via write_file, `--print-timeout 8m`) | Agent v1 importé (`e27c4f3`), fonctionnel la veille |

---

### Task 1 : Contrat v2 — schéma 3 états + mission commune

**Files:**
- Move: `skills/devil-spec/scripts/spec-review-schema.json` → `scripts/spec-review-schema.json`
- Create: `scripts/devil-mission.md`

**Interfaces:**
- Produces: `scripts/spec-review-schema.json` (verdict enum `["approve","rework","reject"]`) et `scripts/devil-mission.md`, consommés par les 3 agents (Tasks 4-6) via `SCHEMA_FILE` / `MISSION_FILE`.

- [ ] **Step 1 : Déplacer le schéma à la racine du plugin**

```bash
mkdir -p scripts && git mv skills/devil-spec/scripts/spec-review-schema.json scripts/spec-review-schema.json
```

- [ ] **Step 2 : Passer le verdict à 3 états (2 edits exacts)**

Dans `scripts/spec-review-schema.json`, Edit 1 :

```json
      "enum": ["approve", "rework"],
```
devient
```json
      "enum": ["approve", "rework", "reject"],
```

Edit 2 :
```json
      "description": "approve si score >= 80, rework sinon"
```
devient
```json
      "description": "approve si score >= 80, rework si 50-79, reject si < 50"
```

- [ ] **Step 3 : Valider le schéma**

```bash
command jq -e '.properties.verdict.enum == ["approve","rework","reject"]' scripts/spec-review-schema.json
```
Attendu : `true`

- [ ] **Step 4 : Créer `scripts/devil-mission.md`** (contenu intégral) :

```markdown
Tu es un architecte logiciel senior qui joue l'avocat du diable sur des
spécifications techniques. Ton travail est de chercher ce qui cloche, pas de
complimenter. Tu compares des SPECS techniques à leur BRAINSTORM d'origine
(les deux documents te sont fournis ci-dessous, en contenu ou en chemins de
fichiers à lire intégralement).

Évalue selon ces 6 critères, chacun scoré de 0 à 100 :

1. **fidelity** — Les specs traduisent-elles correctement l'intention et les
   idées du brainstorm ? Éléments ignorés ou mal interprétés ? Ajouts non
   demandés (scope creep) ?
2. **completeness** — Manque-t-il des sections critiques ? (API, modèle de
   données, gestion d'erreurs, sécurité, tests, déploiement)
3. **consistency** — Y a-t-il des contradictions internes dans les specs ?
4. **feasibility** — Les choix techniques sont-ils réalistes et maintenables ?
5. **security** — Failles ou oublis de sécurité évidents ?
6. **clarity** — Un développeur peut-il implémenter sans ambiguïté ?

Règles :
- score global >= 80 → verdict "approve" ; 50 à 79 → "rework" ; < 50 → "reject"
- "reject" signifie : la spec trahit le brainstorm ou repose sur des choix
  indéfendables ; la réécrire coûte moins cher que la corriger
- Sois exigeant mais constructif : chaque issue DOIT avoir une suggestion
  actionnable
- Si le document est excellent, dis-le (le tableau issues peut être vide)
- Compare systématiquement les specs au brainstorm : dérives, oublis, ajouts
  non demandés
```

- [ ] **Step 5 : Commit**

```bash
git add scripts skills && git commit -m "feat: contrat v2 (schéma 3 états + mission commune)"
```
(+ trailers des Global Constraints — idem pour tous les commits suivants.)

---

### Task 2 : Fixtures `examples/`

**Files:**
- Create: `examples/brainstorming.md`
- Create: `examples/specs.md`

**Interfaces:**
- Produces: le couple de fixtures consommé par les smoke tests (Tasks 4-6, 9). Défauts plantés (oracle, NE PAS écrire dans les fixtures) : (1) fidelity : backend SaaS + comptes = ajout non demandé qui viole « aucun cloud » et les non-objectifs ; (2) fidelity/completeness : export hebdo markdown absent ; (3) security + fidelity : API OpenAI distante + clé en clair, viole « LLM local » ; (4) consistency : SQLite vs « fichiers JSON plats » ; (5) clarity : notifications vagues ; (6) completeness : ni tests ni gestion d'erreurs.

- [ ] **Step 1 : Créer `examples/brainstorming.md`** (contenu intégral) :

```markdown
# Brainstorming — veilleur, CLI de veille RSS locale

## Intention

Un outil CLI personnel qui agrège mes flux RSS techniques et produit un résumé
quotidien lisible en 2 minutes. Local d'abord : AUCUNE donnée envoyée dans le
cloud, c'est non négociable (les flux lus dessinent un profil intellectuel).

## Idées retenues

- CLI TypeScript exécutée via bun, installable en binaire unique.
- Config déclarative en un fichier TOML : liste des flux, cadence, mots-clés.
- Résumés générés par un LLM LOCAL (ollama), jamais par une API distante.
- Notification macOS native quand le digest du jour est prêt.
- Export hebdomadaire en markdown dans un dossier choisi (archive perso).
- Stockage minimal : fichiers plats, pas de serveur, pas de daemon.

## Non-objectifs

- Pas de multi-utilisateurs, pas de comptes, pas de sync entre machines.
- Pas d'interface web.

## Critères de succès

- `veilleur digest` produit le résumé du jour en moins de 30 s sur un M1.
- Zéro requête réseau sortante hors fetch des flux RSS eux-mêmes.
- L'export hebdo est du markdown propre, lisible dans Obsidian.
```

- [ ] **Step 2 : Créer `examples/specs.md`** (contenu intégral — les défauts sont volontaires, aucune annotation ne doit les signaler) :

```markdown
# Specs techniques — veilleur v1

## Architecture

- CLI TypeScript (bun) qui dialogue avec un backend SaaS multi-tenant hébergé
  sur Fly.io.
- Les comptes utilisateurs (email + mot de passe) sont gérés par le backend ;
  chaque machine synchronise ses flux via l'API du backend.
- Le stockage local utilise des fichiers JSON plats dans `~/.veilleur/`.

## Résumés

- Les résumés sont générés par l'API OpenAI (gpt-4o-mini).
- La clé API OpenAI est stockée en clair dans `config.toml`.

## Modèle de données

- Les articles sont persistés dans une base SQLite locale.
- Schéma : `feeds(id, url, tags)`, `articles(id, feed_id, title, url,
  published_at, summary)`.

## Config

- Fichier TOML : liste des flux, cadence de fetch, mots-clés de filtrage.

## Notifications

- Le système notifie l'utilisateur au bon moment.

## Commandes

- `veilleur digest` : produit le résumé du jour.
- `veilleur add <url>` : ajoute un flux.
```

- [ ] **Step 3 : Commit**

```bash
git add examples && git commit -m "feat: fixtures examples pour smoke des devils"
```

---

### Task 3 : Manifest `plugin.json` + README

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `README.md`

**Interfaces:**
- Produces: identité plugin `devil` ; agents et skills découverts via `./agents/` et `./skills/` (déclarés explicitement, style maison caserne).

- [ ] **Step 1 : Créer `.claude-plugin/plugin.json`** (contenu intégral, champs calqués sur l'échantillon caserne 2.8.0) :

```json
{
  "$schema": "https://www.schemastore.org/claude-code-plugin-manifest.json",
  "name": "devil",
  "description": "Avocats du diable pour specs techniques : reviews critiques par Gemini, GLM et Deepseek, unitaires ou en swarm avec verdict consolidé.",
  "version": "0.1.0",
  "author": {
    "name": "Romain Ecarnot",
    "url": "https://github.com/eRom"
  },
  "repository": "https://github.com/eRom/erom-agence-devil",
  "license": "MIT",
  "keywords": ["review", "specs", "devils-advocate", "gemini", "glm", "deepseek", "swarm"],
  "skills": "./skills/",
  "agents": "./agents/"
}
```

- [ ] **Step 2 : Valider**

```bash
command jq -e '.name == "devil" and .version == "0.1.0"' .claude-plugin/plugin.json
```
Attendu : `true`

- [ ] **Step 3 : Créer `README.md`** (contenu intégral) :

```markdown
# devil — avocats du diable pour specs techniques

Plugin Claude Code. Trois reviewers critiques externes jugent une spec
technique contre son brainstorm d'origine : dérives, manques, incohérences,
AVANT implémentation.

## Les devils

| Devil | Modèle | Transport |
|---|---|---|
| gemini | Gemini 3.5 Flash (High) | Antigravity CLI (agy) |
| glm | glm-5.2:cloud | claude CLI → ollama cloud |
| deepseek | deepseek-v4-pro:cloud | claude CLI → ollama cloud |

## Usage

```
/devil-spec                          # auto-detect dans .specs/, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
```

Entrées : toujours 2 fichiers (brainstorming + specs). Sortie par devil :
JSON strict (score 0-100, verdict approve/rework/reject, 6 critères, issues
actionnables). Le swarm consolide : VALABLE, MODIFICATIONS REQUISES, ou
JETABLE, avec convergence des issues (3/3, 2/3, 1/3) et voix dissonantes.

## Prérequis

- `agy` (Antigravity CLI) authentifié, pour le devil gemini.
- `claude` CLI + ollama local avec accès aux modèles cloud (`glm-5.2:cloud`,
  `deepseek-v4-pro:cloud`), pour glm et deepseek.
- `jq`, `trash`.

## Installation

Via la marketplace eRom : `claude plugin install devil@erom-marketplace`.
```

- [ ] **Step 4 : Commit**

```bash
git add .claude-plugin README.md && git commit -m "feat: manifest plugin devil + README"
```

---

### Task 4 : Agent `devil-spec-gemini` (v2 du reviewer importé)

**Files:**
- Move: `agents/devil-spec-reviewer.md` → `agents/devil-spec-gemini.md` (puis réécriture complète)

**Interfaces:**
- Consumes: `scripts/spec-review-schema.json`, `scripts/devil-mission.md` (Task 1), fixtures (Task 2).
- Produces: agent `devil-spec-gemini` ; entrée prompt `BRAINSTORM_FILE=… SPECS_FILE=… SCHEMA_FILE=… MISSION_FILE=…` ; sortie = enveloppe une ligne JSON `{"devil":"gemini","model":"Gemini 3.5 Flash (High)","status":"ok","review":{…}}` ou `{"devil":"gemini","model":"…","status":"error","error":"CLI_FAILED|PARSE_ERROR|TIMEOUT","detail":"…"}`. Contrat identique aux Tasks 5-6.

- [ ] **Step 1 : Renommer**

```bash
git mv agents/devil-spec-reviewer.md agents/devil-spec-gemini.md
```

- [ ] **Step 2 : Réécrire `agents/devil-spec-gemini.md`** (contenu intégral ; la mécanique agy v1 est conservée, seuls l'interface, la mission externalisée et l'enveloppe changent) :

````markdown
---
name: devil-spec-gemini
description: Avocat du diable Gemini — review de specs techniques via Antigravity CLI (agy). Assemble le prompt, appelle Gemini, retourne l'enveloppe JSON.
color: red
tools: Bash, Read, Glob, Grep
model: sonnet
---

Tu es le wrapper du devil Gemini pour `/devil-spec` et `/devil-spec-swarm`.

## Entrée (fournie dans ton prompt)

```
BRAINSTORM_FILE=…
SPECS_FILE=…
SCHEMA_FILE=…
MISSION_FILE=…
```

## Sortie (contrat strict)

Ton message final est UN objet JSON sur une ligne, rien d'autre :
- succès : `{"devil":"gemini","model":"Gemini 3.5 Flash (High)","status":"ok","review":{…}}`
- échec  : `{"devil":"gemini","model":"Gemini 3.5 Flash (High)","status":"error","error":"CLI_FAILED|PARSE_ERROR|TIMEOUT","detail":"≤ 500 chars"}`

## Procédure

IMPORTANT — passe un `timeout` explicite de **540000** ms (9 min) à ton appel
Bash qui lance agy : le défaut de 2 min couperait le run. Côté agy,
`--print-timeout 8m` fait le plafond en dessous. Jamais de rm : `trash`.

Contrat d'invocation agy (appris du plugin antigravity-plugin-cc) :
- `--print` est le DERNIER flag avant le prompt (le parseur Go consomme le token suivant).
- `< /dev/null` après le prompt est obligatoire : sans lui agy peut bloquer hors TTY.
- La review est ÉCRITE dans un fichier par agy (write_file), jamais lue depuis
  stdout : le stdout de `agy --print` peut revenir vide selon les versions
  (bug amont #76) alors que le modèle a répondu.

### Step 1 — Résoudre et assembler le prompt

```bash
BRAINSTORM_ABS=$(realpath "${BRAINSTORM_FILE}")
SPECS_ABS=$(realpath "${SPECS_FILE}")
SCHEMA=$(tr -d '\n' < "${SCHEMA_FILE}")
TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/devil-gemini-XXXXXX")
OUT_FILE="$TMP_DIR/review.json"
PROMPT_FILE="$TMP_DIR/prompt.txt"
{
  cat "${MISSION_FILE}"
  printf '\nTu as accès à deux fichiers :\n- BRAINSTORM : %s\n- SPECS : %s\nLis-les intégralement avant de juger.\n' "$BRAINSTORM_ABS" "$SPECS_ABS"
  printf '\nSortie STRICTEMENT conforme à ce schéma JSON : %s\n' "$SCHEMA"
  printf 'OUTPUT (CRITIQUE) : ne mets PAS la review dans le chat. Écris le seul objet JSON brut via le tool write_file dans : %s\nAprès écriture, confirme uniquement le chemin. C est ton seul livrable.\n' "$OUT_FILE"
} > "$PROMPT_FILE"
```

### Step 2 — Appeler agy (timeout Bash 540000)

```bash
agy --dangerously-skip-permissions \
  --add-dir "$(dirname "$BRAINSTORM_ABS")" --add-dir "$(dirname "$SPECS_ABS")" --add-dir "$TMP_DIR" \
  --model 'Gemini 3.5 Flash (High)' --print-timeout 8m \
  --print "$(cat "$PROMPT_FILE")" < /dev/null
```

### Step 3 — Parser et valider

```bash
REVIEW=$(command jq -c . "$OUT_FILE" 2>/dev/null)
printf '%s' "$REVIEW" | command jq -e 'has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.verdict | IN("approve","rework","reject"))' >/dev/null 2>&1 && VALID=yes || VALID=no
```

Si `REVIEW` est vide ou `VALID=no` : fais UN retry complet (Step 2 puis
Step 3, même OUT_FILE). Toujours en échec après retry :
- `OUT_FILE` absent ou vide → error `CLI_FAILED` (detail : 500 premiers chars
  du stdout agy capturé)
- fichier présent mais JSON invalide → error `PARSE_ERROR` (detail : 500
  premiers chars de OUT_FILE)
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

- [ ] **Step 3 : Smoke test Gemini sur les fixtures** (exécuter ici, dans la session, les Steps 1-4 de l'agent avec `BRAINSTORM_FILE=examples/brainstorming.md SPECS_FILE=examples/specs.md SCHEMA_FILE=scripts/spec-review-schema.json MISSION_FILE=scripts/devil-mission.md`, timeout Bash 540000)

Attendu : enveloppe `status:"ok"`, `review.verdict` ∈ {rework, reject}, `review.issues` non vide. À l'œil : au moins la dérive SaaS ou l'omission de l'export hebdo détectée. Si le verdict est `approve` malgré les 6 défauts plantés : durcir `scripts/devil-mission.md` (un aller-retour max) et re-smoker ; sinon signaler à la review de task.

- [ ] **Step 4 : Commit**

```bash
git add agents && git commit -m "feat: agent devil-spec-gemini (contrat v2, mission externalisée)"
```

---

### Task 5 : Agent `devil-spec-glm`

**Files:**
- Create: `agents/devil-spec-glm.md`

**Interfaces:**
- Consumes: contrat v2 (Task 1), fixtures (Task 2).
- Produces: agent `devil-spec-glm`, même interface et même enveloppe que Task 4 (`"devil":"glm"`, `"model":"glm-5.2:cloud"`).

- [ ] **Step 1 : Créer `agents/devil-spec-glm.md`** (contenu intégral) :

````markdown
---
name: devil-spec-glm
description: Avocat du diable GLM — review de specs techniques via claude CLI sur ollama cloud (glm-5.2:cloud). Assemble le prompt, appelle le modèle, retourne l'enveloppe JSON.
color: red
tools: Bash, Read, Glob, Grep
model: sonnet
---

Tu es le wrapper du devil GLM pour `/devil-spec` et `/devil-spec-swarm`.

## Entrée (fournie dans ton prompt)

```
BRAINSTORM_FILE=…
SPECS_FILE=…
SCHEMA_FILE=…
MISSION_FILE=…
```

## Sortie (contrat strict)

Ton message final est UN objet JSON sur une ligne, rien d'autre :
- succès : `{"devil":"glm","model":"glm-5.2:cloud","status":"ok","review":{…}}`
- échec  : `{"devil":"glm","model":"glm-5.2:cloud","status":"error","error":"CLI_FAILED|PARSE_ERROR|TIMEOUT","detail":"≤ 500 chars"}`

## Procédure

IMPORTANT — passe un `timeout` explicite de **540000** ms (9 min) à chaque
appel Bash qui lance le modèle : le défaut de 2 min couperait le run.
Jamais de `ollama pull`, jamais de préflight de présence du modèle (les
modèles `:cloud` ne sont pas listés par `/api/tags`, c'est normal et validé).
Jamais de rm : `trash`.

### Step 1 — Résoudre et assembler le prompt

```bash
BRAINSTORM_ABS=$(realpath "${BRAINSTORM_FILE}")
SPECS_ABS=$(realpath "${SPECS_FILE}")
SCHEMA=$(tr -d '\n' < "${SCHEMA_FILE}")
TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/devil-glm-XXXXXX")
PROMPT_FILE="$TMP_DIR/prompt.txt"
{
  cat "${MISSION_FILE}"
  printf '\n=== BEGIN BRAINSTORM ===\n'
  cat "$BRAINSTORM_ABS"
  printf '\n=== END BRAINSTORM ===\n\n=== BEGIN SPECS ===\n'
  cat "$SPECS_ABS"
  printf '\n=== END SPECS ===\n'
  printf '\nSortie STRICTEMENT conforme à ce schéma JSON : %s\n' "$SCHEMA"
  printf 'OUTPUT : imprime UNIQUEMENT le seul objet JSON brut de ta review. Pas de fence markdown, pas de texte avant ou après.\n'
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
printf '%s' "$REVIEW" | command jq -e 'has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.verdict | IN("approve","rework","reject"))' >/dev/null 2>&1 && VALID=yes || VALID=no
```

Si `RAW` est vide, ou `IS_ERR` vaut `true`, ou `VALID=no` : fais UN retry
complet (Step 2 puis Step 3). Toujours en échec après retry :
- `RAW` vide ou `IS_ERR=true` → error `CLI_FAILED` (detail : 500 premiers
  chars de `stderr.log`, sinon de RAW)
- sinon → error `PARSE_ERROR` (detail : 500 premiers chars de RESULT)
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

- [ ] **Step 2 : Smoke test GLM sur les fixtures** (exécuter les Steps 1-4 de l'agent ici, mêmes variables que Task 4 Step 3, timeout Bash 540000)

Attendu : enveloppe `status:"ok"`, verdict ∈ {rework, reject}, issues non vide, dérive SaaS ou export manquant détecté à l'œil. Premier run cloud réel : si latence > 9 min systématique, remonter à la review de task au lieu de bricoler.

- [ ] **Step 3 : Commit**

```bash
git add agents/devil-spec-glm.md && git commit -m "feat: agent devil-spec-glm"
```

---

### Task 6 : Agent `devil-spec-deepseek`

**Files:**
- Create: `agents/devil-spec-deepseek.md` (dérivation mécanique de l'agent glm)

**Interfaces:**
- Consumes: `agents/devil-spec-glm.md` committé (Task 5).
- Produces: agent `devil-spec-deepseek`, même contrat (`"devil":"deepseek"`, `"model":"deepseek-v4-pro:cloud"`).

- [ ] **Step 1 : Dériver le fichier** (l'ordre des règles sed est significatif : le modèle d'abord, sinon `s/glm/deepseek/` produirait `deepseek-5.2:cloud`) :

```bash
sed -e 's/glm-5\.2:cloud/deepseek-v4-pro:cloud/g' \
    -e 's/glm/deepseek/g' \
    -e 's/GLM/Deepseek/g' \
    agents/devil-spec-glm.md > agents/devil-spec-deepseek.md
```

- [ ] **Step 2 : Vérifier la dérivation**

```bash
command grep -c 'glm' agents/devil-spec-deepseek.md; command grep -c 'deepseek-v4-pro:cloud' agents/devil-spec-deepseek.md; command grep -n 'name:' agents/devil-spec-deepseek.md | head -1
```
Attendu : `0`, puis ≥ `4`, puis `name: devil-spec-deepseek`.

- [ ] **Step 3 : Smoke test Deepseek sur les fixtures** (identique à Task 5 Step 2, avec la procédure du nouveau fichier)

Attendu : enveloppe `status:"ok"`, verdict ∈ {rework, reject}, issues non vide.

- [ ] **Step 4 : Commit**

```bash
git add agents/devil-spec-deepseek.md && git commit -m "feat: agent devil-spec-deepseek"
```

---

### Task 7 : Skill `/devil-spec` v2

**Files:**
- Rewrite: `skills/devil-spec/SKILL.md`

**Interfaces:**
- Consumes: agents Tasks 4-6 (`subagent_type` plugin-qualifié `devil:devil-spec-<devil>`), contrat v2.
- Produces: commande `/devil-spec [brainstorm] [specs] [gemini|glm|deepseek]`.

- [ ] **Step 1 : Réécrire `skills/devil-spec/SKILL.md`** (contenu intégral) :

````markdown
---
name: devil-spec
description: "Review critique de specs tech par un avocat du diable au choix (Gemini par défaut, GLM, Deepseek). Compare specs au brainstorm pour dérives/manques/incohérences. Triggers: /devil-spec, 'contre spec', 'review spec', 'critique les specs'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-spec — Review critique de specs par un avocat du diable

Un devil externe (Gemini via agy, ou GLM/Deepseek via claude CLI sur ollama
cloud) juge des specs techniques contre leur brainstorm d'origine. L'agent
wrapper gère l'appel et le parsing ; toi (l'orchestrateur) tu présentes le
rapport et guides les corrections.

## Syntaxe

```
/devil-spec                                    # auto-detect, devil gemini
/devil-spec glm                                # auto-detect, devil glm
/devil-spec brainstorm.md specs.md             # paths explicites, gemini
/devil-spec brainstorm.md specs.md deepseek    # paths + devil
```

## Étape 0 — Résoudre le devil et les chemins du plugin

- Devil : le dernier argument s'il vaut `gemini`, `glm` ou `deepseek` ; sinon
  `gemini` par défaut.
- Racine du plugin : deux niveaux au-dessus du « Base directory for this
  skill » injecté ci-dessus. Résous en absolu :
  - `SCHEMA_FILE` = `<racine>/scripts/spec-review-schema.json`
  - `MISSION_FILE` = `<racine>/scripts/devil-mission.md`
- Vérifie que les deux fichiers existent (Read). S'ils manquent, arrête et
  signale un plugin corrompu.

## Étape 1 — Identifier les fichiers d'entrée

### Si les paths sont fournis en argument
Utilise-les directement. Vérifie qu'ils existent.

### Si aucun path
Auto-detect :
1. Cherche `brainstorming.md` et `*-technique.md` ou `plan.md` dans `.specs/`
   (Glob `**/*.md`)
2. Si plusieurs chantiers existent, demande à Romain lequel reviewer
3. Si un seul chantier, prends le brainstorming.md + le fichier de specs le
   plus récent

### Confirmation
Toujours confirmer avant de lancer :

> **Fichiers détectés :**
> - Brainstorm : `.specs/mvp/brainstorming.md`
> - Specs : `.specs/mvp/architecture-technique.md`
> - Devil : glm (glm-5.2:cloud)
>
> Je lance la review ?

Attends la confirmation de Romain (« oui », « go », « lance »).

## Étape 2 — Lancer le sous-agent

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

Annonce avant le spawn : « **Review en cours…** <devil> analyse les specs vs
le brainstorm (jusqu'à 9 min). »

## Étape 3 — Parser l'enveloppe et présenter le rapport

Le retour de l'agent est UNE ligne JSON : `{devil, model, status,
review|error+detail}`.

### Si `status: "error"`
```
══════ REVIEW ÉCHOUÉE ══════

Devil : <devil> (<model>)
Erreur : <error> — <detail>

→ Relance (/devil-spec <devil>), ou essaie un autre devil
  (/devil-spec glm|deepseek|gemini), ou review manuelle.
```

### Si `status: "ok"`
```
══════ REVIEW DE SPECS ══════

Devil : <devil> (<model>)
Score : <score>/100  [APPROVE ✓ | REWORK ✗ | REJECT ☠]

Critères :
  Fidélité      <n>/100  <commentaire court>
  Complétude    <n>/100  <commentaire court>
  Cohérence     <n>/100  <commentaire court>
  Faisabilité   <n>/100  <commentaire court>
  Sécurité      <n>/100  <commentaire court>
  Clarté        <n>/100  <commentaire court>

Résumé : <summary>
```

Issues (si non vide), tableau trié par sévérité (critical > high > medium > low) :

```
| Sév | Cat | Problème | Suggestion | Source |
```

## Étape 4 — Next steps selon le verdict

### approve (score ≥ 80)
> **Verdict : APPROVE** — Les specs sont solides. [issues mineures s'il y en a]
> → On continue le flow ?

### rework (50-79)
> **Verdict : REWORK** — <n> issues à adresser.
> 1. **Corriger** — j'adresse les issues dans les specs
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-review** — je corrige puis je relance le même devil

### reject (< 50)
> **Verdict : REJECT** — la spec trahit le brainstorm ou repose sur des choix
> indéfendables. Pas de correction incrémentale : on rouvre le brainstorm
> (réviser l'intention, ou retailler le périmètre), puis on réécrit les specs.

Attends la décision de Romain.

## Étape 5 — Correction (si demandée)

1. Lis le fichier de specs complet
2. Adresse chaque issue par sévérité décroissante (critical > high > medium)
3. Explique brièvement chaque changement
4. Écris via Edit — UNIQUEMENT le fichier de specs
5. Résumé des corrections, puis :
   - **Re-review** → relance le MÊME devil, mêmes fichiers → Étape 3
   - **Corriger** seul → demande validation à Romain

## Règles

- Ne JAMAIS modifier le brainstorm — seulement les specs.
- Ne JAMAIS inventer du contenu — corriger sur la base des issues + du brainstorm.
- Les issues `low` ne sont PAS corrigées sauf demande explicite.
- Maximum 2 cycles de re-review ; après 2 rework consécutifs, Romain tranche.
- Le devil ne modifie rien : c'est toi qui corriges.
````

- [ ] **Step 2 : Vérifier l'absence de chemins en dur**

```bash
command grep -rn '~/.claude' skills/ agents/ scripts/ || echo CLEAN
```
Attendu : `CLEAN`

- [ ] **Step 3 : Commit**

```bash
git add skills/devil-spec && git commit -m "feat: skill /devil-spec v2 (choix du devil, contrat v2)"
```

---

### Task 8 : Skill `/devil-spec-swarm`

**Files:**
- Create: `skills/devil-spec-swarm/SKILL.md`

**Interfaces:**
- Consumes: les 3 agents (même spawn que Task 7), enveloppes v2.
- Produces: commande `/devil-spec-swarm [brainstorm] [specs]`.

- [ ] **Step 1 : Créer `skills/devil-spec-swarm/SKILL.md`** (contenu intégral) :

````markdown
---
name: devil-spec-swarm
description: "Tribunal des avocats du diable : Gemini + GLM + Deepseek reviewent les specs en parallèle, synthèse consolidée avec verdict VALABLE / MODIFICATIONS REQUISES / JETABLE. Triggers: /devil-spec-swarm, 'swarm de review', 'tribunal des specs'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-spec-swarm — Le tribunal des trois devils

Les 3 avocats du diable (Gemini via agy, GLM et Deepseek via ollama cloud)
jugent les mêmes specs EN PARALLÈLE. Toi (l'orchestrateur) tu consolides :
convergences, dissonances, verdict final argumenté.

## Étape 0 — Chemins du plugin

Identique à /devil-spec : racine = deux niveaux au-dessus du base directory
injecté ; résous `SCHEMA_FILE` = `<racine>/scripts/spec-review-schema.json`
et `MISSION_FILE` = `<racine>/scripts/devil-mission.md`, vérifie l'existence.

## Étape 1 — Fichiers d'entrée

Même détection, mêmes règles et même confirmation que /devil-spec (paths en
argument sinon auto-detect `.specs/`), avec l'annonce :

> **Tribunal convoqué :** gemini + glm + deepseek en parallèle (jusqu'à 9 min).
> Fichiers : <brainstorm> vs <specs>. Je lance ?

## Étape 2 — Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message (c'est ce qui les
fait tourner en parallèle). Même prompt pour les trois, seul le
subagent_type change : `devil:devil-spec-gemini`, `devil:devil-spec-glm`,
`devil:devil-spec-deepseek` (fallback sans préfixe si non résolu) :

```
prompt: "BRAINSTORM_FILE=<abs>\nSPECS_FILE=<abs>\nSCHEMA_FILE=<abs>\nMISSION_FILE=<abs>\n\nExécute la procédure de review."
```

## Étape 3 — Collecte et quorum

Chaque retour est une enveloppe `{devil, model, status, review|error}`.

- 3 voix `ok` → synthèse pleine.
- 2 voix `ok` → continue, et le rapport OUVRE sur la voix absente :
  « ⚠ <devil> n'a pas rendu son verdict (<error> — <detail court>). Synthèse
  sur 2 voix. »
- ≤ 1 voix `ok` → PAS de verdict. Rapport d'échec avec le détail des erreurs,
  proposer : relancer le swarm, ou basculer en unitaire (/devil-spec <devil>).

Un retour qui n'est pas une enveloppe JSON valide compte comme voix absente
(ne JAMAIS interpréter un texte d'erreur comme une review).

## Étape 4 — Synthèse (ton travail d'orchestrateur)

### Verdict final

| Situation (sur les voix exprimées) | Verdict |
|---|---|
| ≥ 2 approve et 0 reject | **VALABLE** |
| ≥ 2 reject | **JETABLE** |
| tout le reste | **MODIFICATIONS REQUISES** |

### Consolidation des issues

- Regroupe les issues des devils par PROBLÈME DE FOND (même problème ≠ mêmes
  mots — c'est un jugement sémantique, pas un match textuel).
- Badge de convergence : 3/3, 2/3, 1/3 (sur les voix exprimées : à 2 voix,
  2/2 et 1/2).
- Tri : convergence décroissante, puis sévérité max du groupe.
- Garde la suggestion la plus actionnable du groupe ; sévérité = la plus haute.

### Voix dissonante

Si un devil s'écarte des deux autres (ex. reject isolé contre 2 approve, ou
écart de score > 30 sur un critère), section dédiée : qui, sur quoi, son
argument principal. La majorité ne l'écrase JAMAIS silencieusement.

## Étape 5 — Rapport

```
══════ TRIBUNAL DES DEVILS ══════

[⚠ voix absente le cas échéant]

Verdict : [VALABLE ✓ | MODIFICATIONS REQUISES ⚠ | JETABLE ☠]
<2-3 phrases d'argumentation : pourquoi ce verdict, à partir des verdicts
individuels et des convergences>

Scores :
  Critère       gemini  glm  deepseek  (moyenne)
  Fidélité        <n>   <n>    <n>       <n>
  Complétude      <n>   <n>    <n>       <n>
  Cohérence       <n>   <n>    <n>       <n>
  Faisabilité     <n>   <n>    <n>       <n>
  Sécurité        <n>   <n>    <n>       <n>
  Clarté          <n>   <n>    <n>       <n>
  GLOBAL          <n>   <n>    <n>       <n>

Verdicts : gemini <verdict> · glm <verdict> · deepseek <verdict>
```

Issues consolidées (tableau) :

```
| Conv | Sév | Cat | Problème | Suggestion | Devils |
| 3/3  | critical | fidelity | … | … | gemini, glm, deepseek |
```

Puis la section « Voix dissonante » si applicable.

## Étape 6 — Next steps selon le verdict

### VALABLE
> Le tribunal valide. [Mentionner les issues 2/3+ restantes s'il y en a]
> → On continue le flow (plan d'implémentation) ?

### MODIFICATIONS REQUISES
> 1. **Corriger** — j'adresse les issues consolidées, convergentes (3/3, 2/3)
>    d'abord, puis critical/high isolées
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-swarm** — je corrige puis je reconvoque le tribunal (1 max)

### JETABLE
> Le tribunal enterre la spec : <les 1-2 raisons de fond convergentes>.
> Pas de correction incrémentale : on rouvre le brainstorm, puis specs neuves.

## Règles

- Ne JAMAIS modifier le brainstorm.
- Corrections : mêmes règles que /devil-spec (Edit specs uniquement, low
  ignorées, pas d'invention).
- Maximum 1 re-swarm ; ensuite Romain tranche.
- Coût : 3 modèles en parallèle ≈ la durée du plus lent. C'est un gate de
  qualité, pas un lint : ne pas le déclencher sur un brouillon.
````

- [ ] **Step 2 : Vérification structurelle**

```bash
command grep -rn '~/.claude' skills/devil-spec-swarm/ || echo CLEAN; command grep -c 'devil:devil-spec' skills/devil-spec-swarm/SKILL.md
```
Attendu : `CLEAN`, puis ≥ `1`.

- [ ] **Step 3 : Commit**

```bash
git add skills/devil-spec-swarm && git commit -m "feat: skill /devil-spec-swarm (tribunal + synthèse)"
```

---

### Task 9 : Validation du chargement plugin + chemin d'erreur

**Files:** aucun (tâche de vérification — pas de commit attendu, sauf correctifs révélés).

**Interfaces:**
- Consumes: tout le plugin (Tasks 1-8).

- [ ] **Step 1 : Capturer les options de `claude plugin details`**

```bash
command claude plugin details --help 2>&1 | head -20
```

- [ ] **Step 2 : Inventaire du plugin depuis le repo local** (non-normatif : ajuster la forme à ce que le Step 1 a montré ; forme probable `claude --plugin-dir /Users/recarnot/dev/erom-agence-devil plugin details devil`)

Contrat de succès : l'inventaire liste 2 skills (devil-spec, devil-spec-swarm) et 3 agents (devil-spec-gemini, devil-spec-glm, devil-spec-deepseek), zéro erreur de parsing. En échec de forme CLI : fallback `claude --plugin-dir . plugin list`. Si les agents n'apparaissent pas à cause de la clé `"agents"` du manifest : retirer la clé (découverte implicite du dossier `agents/`) et re-tester.

- [ ] **Step 3 : Chemin d'erreur — modèle volontairement inexistant** (teste NOTRE gestion d'erreur, pas la connectivité validée par Romain)

Exécuter les Steps 1-4 de l'agent glm sur les fixtures en remplaçant, dans le Step 2 seulement, `--model glm-5.2:cloud` par `--model glm-typo:cloud` (timeout Bash 540000).

Attendu : le flux aboutit à une enveloppe `{"devil":"glm","model":"glm-5.2:cloud","status":"error","error":"CLI_FAILED","detail":"…"}` après 1 retry, et `stderr.log`/RAW contient l'erreur amont. Aucun crash, aucune sortie non-JSON.

---

### Task 10 : Entrée marketplace (repo erom-marketplace)

**Files:**
- Modify: `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json`

**Interfaces:**
- Consumes: plugin complet (Tasks 1-9).
- Produces: plugin `devil` installable via `devil@erom-marketplace` (après push des 2 repos — gated Romain).

- [ ] **Step 1 : Inspecter l'état git du repo marketplace**

```bash
command git -C /Users/recarnot/dev/erom-marketplace branch --show-current && command git -C /Users/recarnot/dev/erom-marketplace status --short
```
Attendu : `main`, arbre propre. Sinon : signaler et NE PAS committer par-dessus.

- [ ] **Step 2 : Ajouter l'entrée plugin** — dans `marketplace.json`, Edit :

```json
      "version": "2.8.0",
      "strict": true
    }
  ]
```
devient
```json
      "version": "2.8.0",
      "strict": true
    },
    {
      "name": "devil",
      "source": {
        "source": "url",
        "url": "https://github.com/eRom/erom-agence-devil.git"
      },
      "description": "Avocats du diable pour specs techniques : Gemini, GLM et Deepseek, unitaires ou en swarm avec verdict consolidé.",
      "version": "0.1.0",
      "strict": true
    }
  ]
```

- [ ] **Step 3 : Bump la version marketplace** — Edit :

```json
    "version": "0.4.0"
```
devient
```json
    "version": "0.5.0"
```

- [ ] **Step 4 : Valider et committer (dans erom-marketplace)**

```bash
command jq -e '.plugins | length == 2 and (.[1].name == "devil")' /Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json
git -C /Users/recarnot/dev/erom-marketplace add .claude-plugin/marketplace.json
git -C /Users/recarnot/dev/erom-marketplace commit -m "feat: plugin devil 0.1.0 (avocats du diable specs)"
```
Attendu jq : `true`. PAS de push (gate Romain, voir clôture).

---

## Clôture (gated Romain — jamais autonome)

À présenter en un bloc à la fin, chaque étape à sa main :

1. **Dogfood** : session interactive `cd /Users/recarnot/dev/erom-agence-devil && claude --plugin-dir .`, d'abord `/devil-spec-swarm examples/brainstorming.md examples/specs.md` (couvre spec §7.3 : 3 voix, synthèse, verdict sur fixtures — attendu MODIFICATIONS REQUISES ou JETABLE vu les 6 défauts plantés), puis `/devil-spec-swarm .specs/plugin-devil/brainstorming.md .specs/plugin-devil/architecture-technique.md` — le tribunal juge sa propre spec. C'est le test d'acceptation.
2. **Push** `erom-agence-devil` (origin déjà configuré) puis **push** `erom-marketplace` — requis pour l'install par URL.
3. **Bascule** : `trash ~/.claude/agents/devil-spec-reviewer.md` et `trash ~/.claude/skills/devil-spec/` pour tuer la collision `/devil-spec`, puis `claude plugin install devil@erom-marketplace`.
4. Retirer `examples/` ? Non — ils servent de démo et de fixtures de re-test. Garder.
