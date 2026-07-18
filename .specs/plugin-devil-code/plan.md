# devil code v0.3.0 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ajouter la review critique de code `/devil-code` + `/devil-code-swarm` au plugin devil : un changement (PR, branche, range, working tree) jugé par les devils externes, agents transport inchangés.

**Architecture:** Pattern v0.2.0 pur — 1 mission + 1 schéma + 2 skills, zéro modification des 4 agents. La skill orchestratrice résout le diff, packe DIFF + FILES + INTENT en fichiers temp, scanne les secrets AVANT envoi, spawne le(s) transport(s), vérifie l'ancrage file:ligne au retour, restitue score/verdict/issues. Référence : `.specs/plugin-devil-code/{brainstorming,architecture-technique}.md` (post-tribunal, commit 0a641ee).

**Tech Stack:** Markdown (skills Claude Code plugin), bash + `git` + `gh` + `jq` + `grep -E`, agents transport existants (`agy` Gemini, `claude -p` → ollama cloud GLM/Deepseek, `claude` Opus), `trash`.

## Global Constraints

- Jamais `rm` / `rmdir` : suppression via `trash` uniquement.
- Timeout Bash explicite **540000** ms + **1 retry** sur tout appel modèle.
- Jamais `ollama pull`, jamais de préflight `/api/tags` (modèles `:cloud` non listés, normal et validé).
- Les 4 agents (`agents/devil-{gemini,glm,deepseek,opus}.md`) sont INTOUCHÉS : toute tâche qui voudrait les modifier est hors périmètre.
- Surfaces `/devil-spec*` et `/devil-brain*` : INCHANGÉES (aucun fichier de ces exercices n'est touché).
- Enums JSON en anglais, contenus rédigés en français.
- Zéro chemin `~/.claude/` en dur dans `skills/` et `scripts/`.
- VALIDATE_JQ code : BYTE-IDENTIQUE entre `skills/devil-code/SKILL.md` et `skills/devil-code-swarm/SKILL.md` (risque n°1 de désync des jumelles).
- Manifest plugin : PAS de clé `agents` (rejetée par le schéma, auto-découverte) ; `--plugin-dir` est un flag GLOBAL (avant la sous-commande `plugin`).
- MàJ plugin installé = `uninstall` PUIS `install` (install déjà-installé est no-op, update nom-nu échoue — gotcha mémoire).
- Sorties de commande pilotant une décision : préfixer `command` (hook rtk réécrit git/grep/ls).
- Greps de non-présence : exit 1 = SUCCÈS attendu (0 occurrence), jamais un échec de tâche.
- Commits : trailers `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` + `Claude-Session: https://claude.ai/code/session_01DBfvHA9RHw9B5XYq7DdiGt`.
- Fixtures `examples/code-*` : inputs DIRECTS d'agent (paquet pré-assemblé), jamais générées via git au moment du smoke.

## Contrat de spawn agent (interface commune, référencée par les Tasks 3, 4, 6, 7)

Prompt de spawn d'un agent devil (texte, lignes exactes) :

```
MISSION_FILE=<abs>
SCHEMA_FILE=<abs>
VALIDATE_JQ=<expression jq -e sur une ligne>
INPUTS:
DIFF:<abs>
[FILES:<abs>]
[INTENT:<abs>]

Exécute la procédure de transport.
```

Expression VALIDATE_JQ code (UNE ligne ; l'agent la pose en bash entre single quotes). **Testée le 2026-07-18 sur jq 1.8.1, 3 échantillons : conforme → pass ; issue sans `failure_scenario` → fail ; critère score 150 ou remplacé par une string → fail** :

```
has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.score|type=="number" and .>=0 and .<=100) and (.verdict|IN("approve","rework","reject")) and (.criteria|has("correctness") and has("architecture") and has("security") and has("performance") and has("tests") and has("maintainability")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type=="object" and has("score") and has("comment") and (.score|type=="number" and .>=0 and .<=100))) and (.issues|type=="array" and all(has("severity") and has("category") and has("file") and has("description") and has("failure_scenario") and has("suggestion") and (.severity|IN("critical","high","medium","low")) and (.category|IN("correctness","architecture","security","performance","tests","maintainability","intent"))))
```

Enveloppe de retour (contrat strict, une ligne) : `{"devil":"<d>","model":"<m>","status":"ok","review":{…}}` ou `{"devil":"<d>","model":"<m>","status":"error","error":"CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT","detail":"≤ 500 chars"}`.

Convention externe capturée : les commandes `gh pr view --json title,body,author,baseRefName,headRefName,state,additions,deletions,changedFiles` et `gh pr diff <n>` proviennent du prompt du `/review` built-in extrait du binaire claude 2.1.214 (session 2026-07-18) — pas de mémoire, un échantillon réel.

---

### Task 1: Contrat code (scripts/)

**Files:**
- Create: `scripts/devil-code-mission.md`
- Create: `scripts/devil-code-schema.json`

**Interfaces:**
- Produces: les 2 chemins ci-dessus, consommés par les skills (Tasks 3, 4) via `<racine plugin>/scripts/<fichier>` ; la VALIDATE_JQ du header (déjà testée) les accompagne.

- [ ] **Step 1: Créer `scripts/devil-code-mission.md`** (contenu complet)

```markdown
Tu es un reviewer senior qui joue l'avocat du diable sur un CHANGEMENT de
code. Ton travail est de chercher ce qui cloche dans ce changement, pas de
complimenter — et pas de juger le reste du dépôt.

Tes entrées, chacune entre marqueurs === BEGIN/END LABEL === :

- DIFF — le changement (diff unifié). C'est le SEUL périmètre des issues :
  interdit de flagger du code pré-existant que le diff ne touche pas.
- FILES (optionnel) — l'état final des fichiers modifiés, concaténés sous
  des en-têtes `=== FILE: chemin ===`. C'est du CONTEXTE pour comprendre le
  diff. S'il est ABSENT, tu review sur diff seul : calibre — pour toute
  issue qui dépend d'un contexte que tu n'as pas, choisis une sévérité
  prudente et nomme la dépendance manquante dans failure_scenario.
- INTENT (optionnel) — l'intention du changement (spec, plan ou description
  de PR). S'il est fourni, juge AUSSI l'alignement du code sur l'intention
  (catégorie `intent` : dérives, oublis, ajouts non demandés). S'il est
  absent, la catégorie `intent` est INTERDITE.

Évalue selon ces 6 critères, chacun scoré de 0 à 100 :

1. **correctness** — bugs runtime introduits par le changement : condition
   inversée, off-by-one, null/undefined déréférencé, guard supprimé, await
   manquant, erreur avalée, copy-paste de mauvaise variable.
2. **architecture** — design et intégration : couplage, duplication d'un
   helper visible dans les entrées, responsabilité au mauvais endroit.
3. **security** — injections, authz manquante, secrets en dur, validation
   des entrées aux frontières réelles.
4. **performance** — complexité, requête dans une boucle (N+1), travail
   inutile dans un chemin chaud visible.
5. **tests** — le changement est-il testé ? Les tests vérifient-ils du
   comportement réel (assertions significatives) ?
6. **maintainability** — lisibilité réelle, conventions du fichier hôte,
   gestion d'erreurs.

Règles dures (anti-bruit) :
- Chaque issue DOIT porter : `file` au format `chemin:ligne` (ligne de
  l'état FINAL du fichier), un `failure_scenario`, une `suggestion`
  actionnable.
- `failure_scenario` = la conséquence concrète et située :
  - correctness / security / performance / tests → des entrées concrètes
    et le résultat faux qu'elles produisent ;
  - architecture / maintainability → le coût futur nommé et son
    déclencheur (« ajouter un 4e transport forcera à dupliquer X dans
    3 skills »).
  Pas de conséquence concrète nommable = PAS d'issue.
- Style et naming purs INTERDITS (aucune issue « renommer », « reformater »).
- Ne flagge qu'à confiance élevée : uniquement ce que tu peux défendre.
- Les fichiers de test modifiés se jugent au titre du critère `tests`, pas
  comme du code de production.
- Si le changement est excellent, dis-le : `issues` peut être vide.

Verdict : score >= 80 → "approve" ; 50 à 79 → "rework" ; < 50 → "reject".
"reject" signifie : le changement est structurellement mauvais, le réécrire
coûte moins cher que le corriger.
```

- [ ] **Step 2: Créer `scripts/devil-code-schema.json`** (contenu complet)

```json
{
  "type": "object",
  "required": ["score", "verdict", "summary", "criteria", "issues"],
  "properties": {
    "score": {
      "type": "integer",
      "minimum": 0,
      "maximum": 100,
      "description": "Score global du changement (0-100)"
    },
    "verdict": {
      "type": "string",
      "enum": ["approve", "rework", "reject"],
      "description": "approve si score >= 80, rework si 50-79, reject si < 50"
    },
    "summary": {
      "type": "string",
      "description": "Résumé en 2-3 phrases de l'évaluation globale"
    },
    "criteria": {
      "type": "object",
      "required": ["correctness", "architecture", "security", "performance", "tests", "maintainability"],
      "properties": {
        "correctness": {
          "type": "object",
          "required": ["score", "comment"],
          "properties": {
            "score": { "type": "integer", "minimum": 0, "maximum": 100 },
            "comment": { "type": "string" }
          }
        },
        "architecture": {
          "type": "object",
          "required": ["score", "comment"],
          "properties": {
            "score": { "type": "integer", "minimum": 0, "maximum": 100 },
            "comment": { "type": "string" }
          }
        },
        "security": {
          "type": "object",
          "required": ["score", "comment"],
          "properties": {
            "score": { "type": "integer", "minimum": 0, "maximum": 100 },
            "comment": { "type": "string" }
          }
        },
        "performance": {
          "type": "object",
          "required": ["score", "comment"],
          "properties": {
            "score": { "type": "integer", "minimum": 0, "maximum": 100 },
            "comment": { "type": "string" }
          }
        },
        "tests": {
          "type": "object",
          "required": ["score", "comment"],
          "properties": {
            "score": { "type": "integer", "minimum": 0, "maximum": 100 },
            "comment": { "type": "string" }
          }
        },
        "maintainability": {
          "type": "object",
          "required": ["score", "comment"],
          "properties": {
            "score": { "type": "integer", "minimum": 0, "maximum": 100 },
            "comment": { "type": "string" }
          }
        }
      }
    },
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["severity", "category", "file", "description", "failure_scenario", "suggestion"],
        "properties": {
          "severity": {
            "type": "string",
            "enum": ["critical", "high", "medium", "low"]
          },
          "category": {
            "type": "string",
            "enum": ["correctness", "architecture", "security", "performance", "tests", "maintainability", "intent"]
          },
          "file": {
            "type": "string",
            "description": "chemin/fichier.ext:ligne — ligne de l'état final du fichier"
          },
          "description": {
            "type": "string",
            "description": "Le problème identifié"
          },
          "failure_scenario": {
            "type": "string",
            "description": "Conséquence concrète et située : entrées → résultat faux (runtime), ou coût futur nommé + déclencheur (structurel)"
          },
          "suggestion": {
            "type": "string",
            "description": "Correction actionnable"
          }
        }
      }
    }
  }
}
```

- [ ] **Step 3: Vérifier**

```bash
command jq . scripts/devil-code-schema.json >/dev/null && echo CODE_SCHEMA_OK
ls scripts/
```
Attendu : `CODE_SCHEMA_OK`, et `scripts/` contient exactement 6 fichiers : `devil-spec-{mission.md,schema.json}`, `devil-brain-{mission.md,schema.json}`, `devil-code-{mission.md,schema.json}`.

- [ ] **Step 4: Re-tester la VALIDATE_JQ code contre le schéma livré** (les 3 échantillons du header ; échantillons complets) :

```bash
TMP=$(mktemp -d "${TMPDIR:-/tmp}/devil-code-jq-XXXXXX")
cat > "$TMP/valid.json" <<'EOF'
{"score":72,"verdict":"rework","summary":"ok","criteria":{"correctness":{"score":70,"comment":"c"},"architecture":{"score":80,"comment":"c"},"security":{"score":60,"comment":"c"},"performance":{"score":75,"comment":"c"},"tests":{"score":65,"comment":"c"},"maintainability":{"score":80,"comment":"c"}},"issues":[{"severity":"high","category":"correctness","file":"src/a.ts:42","description":"d","failure_scenario":"f","suggestion":"s"}]}
EOF
cat > "$TMP/invalid.json" <<'EOF'
{"score":72,"verdict":"rework","summary":"ok","criteria":{"correctness":{"score":70,"comment":"c"},"architecture":{"score":80,"comment":"c"},"security":{"score":60,"comment":"c"},"performance":{"score":75,"comment":"c"},"tests":{"score":65,"comment":"c"},"maintainability":{"score":80,"comment":"c"}},"issues":[{"severity":"high","category":"correctness","file":"src/a.ts:42","description":"d","suggestion":"s"}]}
EOF
cat > "$TMP/criteria150.json" <<'EOF'
{"score":72,"verdict":"rework","summary":"ok","criteria":{"correctness":{"score":150,"comment":"c"},"architecture":"garbage","security":{"score":60,"comment":"c"},"performance":{"score":75,"comment":"c"},"tests":{"score":65,"comment":"c"},"maintainability":{"score":80,"comment":"c"}},"issues":[]}
EOF
V='has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.score|type=="number" and .>=0 and .<=100) and (.verdict|IN("approve","rework","reject")) and (.criteria|has("correctness") and has("architecture") and has("security") and has("performance") and has("tests") and has("maintainability")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type=="object" and has("score") and has("comment") and (.score|type=="number" and .>=0 and .<=100))) and (.issues|type=="array" and all(has("severity") and has("category") and has("file") and has("description") and has("failure_scenario") and has("suggestion") and (.severity|IN("critical","high","medium","low")) and (.category|IN("correctness","architecture","security","performance","tests","maintainability","intent"))))'
command jq -e "$V" "$TMP/valid.json" >/dev/null && echo V_PASS || echo V_FAIL
command jq -e "$V" "$TMP/invalid.json" >/dev/null && echo I_PASS || echo I_FAIL
command jq -e "$V" "$TMP/criteria150.json" >/dev/null && echo C_PASS || echo C_FAIL
trash "$TMP"
```
Attendu : `V_PASS`, `I_FAIL`, `C_FAIL`.

- [ ] **Step 5: Commit**

```bash
command git add scripts/
command git commit -m "feat(contracts): mission et schéma de l'exercice code"
```

---

### Task 2: Fixtures plantées (examples/)

**Files:**
- Create: `examples/code-diff.patch`
- Create: `examples/code-files.txt`
- Create: `examples/code-secret.patch`

**Interfaces:**
- Produces: paquet DIFF+FILES pré-assemblé pour le smoke (Task 6) avec 5 défauts plantés — injection SQL `src/db.ts:17` (security, critical attendu), null deref `src/auth.ts:9` (correctness), N+1 `src/report.ts:8` (performance), duplication `maskName` `src/auth.ts:19` / `src/report.ts:14` (architecture ou maintainability), test sans assertion `tests/auth.test.ts:3` (tests) — et fixture secret pour le test du scan pré-vol. `AKIAIOSFODNN7EXAMPLE` et `wJalrXUtnFEMI/…` sont les exemples canoniques de la documentation AWS (échantillon capturé, pas inventé).

- [ ] **Step 1: Créer `examples/code-diff.patch`** (contenu complet ; arithmétique des hunks vérifiée à la main : db.ts `-11,8 +11,14`, nouveaux fichiers `+1,21` / `+1,16` / `+1,5`)

```diff
diff --git a/src/db.ts b/src/db.ts
index 3f1c2aa..9d4e7bb 100644
--- a/src/db.ts
+++ b/src/db.ts
@@ -11,8 +11,14 @@
 export function getUser(id: number): User | null {
   const row = db.prepare("SELECT * FROM users WHERE id = ?").get(id);
   return row ? (row as User) : null;
 }
+
+export function getUserByName(name: string): User | null {
+  const sql = "SELECT * FROM users WHERE name = '" + name + "'";
+  const row = db.prepare(sql).get();
+  return row ? (row as User) : null;
+}
 
 export function getOrders(userId: number): { id: number; total: number }[] {
   return db.prepare("SELECT * FROM orders WHERE user_id = ?").all(userId);
 }
diff --git a/src/auth.ts b/src/auth.ts
new file mode 100644
index 0000000..aa11bb2
--- /dev/null
+++ b/src/auth.ts
@@ -0,0 +1,21 @@
+import crypto from "node:crypto";
+import { getUserByName } from "./db";
+import { issueToken } from "./token";
+import { credentials } from "./store";
+
+export function login(name: string, password: string): string {
+  const user = getUserByName(name);
+  const hash = sha256hex(password);
+  if (hash !== credentials.get(user.id)) {
+    throw new Error(`invalid credentials for ${maskName(name)}`);
+  }
+  return issueToken(user.id, user.role);
+}
+
+function sha256hex(input: string): string {
+  return crypto.createHash("sha256").update(input).digest("hex");
+}
+
+function maskName(name: string): string {
+  return name.charAt(0) + "***";
+}
diff --git a/src/report.ts b/src/report.ts
new file mode 100644
index 0000000..bb22cc3
--- /dev/null
+++ b/src/report.ts
@@ -0,0 +1,16 @@
+import { getUser, getOrders } from "./db";
+
+export function buildReport(userIds: number[]): string[] {
+  const lines: string[] = [];
+  for (const id of userIds) {
+    const user = getUser(id);
+    if (!user) continue;
+    const orders = getOrders(user.id);
+    lines.push(`${maskName(user.name)}: ${orders.length} commandes`);
+  }
+  return lines;
+}
+
+function maskName(name: string): string {
+  return name.charAt(0) + "***";
+}
diff --git a/tests/auth.test.ts b/tests/auth.test.ts
new file mode 100644
index 0000000..cc33dd4
--- /dev/null
+++ b/tests/auth.test.ts
@@ -0,0 +1,5 @@
+import { login } from "../src/auth";
+
+test("login retourne un token", () => {
+  login("alice", "s3cret!");
+});
```

- [ ] **Step 2: Créer `examples/code-files.txt`** (contenu complet — états finaux, cohérents ligne à ligne avec le patch)

```text
=== FILE: src/db.ts ===
import { Database } from "./engine";

export interface User {
  id: number;
  name: string;
  role: string;
}

const db = new Database("app.db");

export function getUser(id: number): User | null {
  const row = db.prepare("SELECT * FROM users WHERE id = ?").get(id);
  return row ? (row as User) : null;
}

export function getUserByName(name: string): User | null {
  const sql = "SELECT * FROM users WHERE name = '" + name + "'";
  const row = db.prepare(sql).get();
  return row ? (row as User) : null;
}

export function getOrders(userId: number): { id: number; total: number }[] {
  return db.prepare("SELECT * FROM orders WHERE user_id = ?").all(userId);
}

=== FILE: src/auth.ts ===
import crypto from "node:crypto";
import { getUserByName } from "./db";
import { issueToken } from "./token";
import { credentials } from "./store";

export function login(name: string, password: string): string {
  const user = getUserByName(name);
  const hash = sha256hex(password);
  if (hash !== credentials.get(user.id)) {
    throw new Error(`invalid credentials for ${maskName(name)}`);
  }
  return issueToken(user.id, user.role);
}

function sha256hex(input: string): string {
  return crypto.createHash("sha256").update(input).digest("hex");
}

function maskName(name: string): string {
  return name.charAt(0) + "***";
}

=== FILE: src/report.ts ===
import { getUser, getOrders } from "./db";

export function buildReport(userIds: number[]): string[] {
  const lines: string[] = [];
  for (const id of userIds) {
    const user = getUser(id);
    if (!user) continue;
    const orders = getOrders(user.id);
    lines.push(`${maskName(user.name)}: ${orders.length} commandes`);
  }
  return lines;
}

function maskName(name: string): string {
  return name.charAt(0) + "***";
}

=== FILE: tests/auth.test.ts ===
import { login } from "../src/auth";

test("login retourne un token", () => {
  login("alice", "s3cret!");
});
```

- [ ] **Step 3: Créer `examples/code-secret.patch`** (contenu complet — clés d'exemple de la documentation AWS)

```diff
diff --git a/config/deploy.ts b/config/deploy.ts
new file mode 100644
index 0000000..1111111
--- /dev/null
+++ b/config/deploy.ts
@@ -0,0 +1,6 @@
+export const deploy = {
+  region: "eu-west-3",
+  accessKeyId: "AKIAIOSFODNN7EXAMPLE",
+  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
+};
+// -----BEGIN RSA PRIVATE KEY----- collée ici par erreur, à retirer
```

- [ ] **Step 4: Vérifier — la fixture code est propre au scan, la fixture secret le déclenche**

```bash
command grep -c '^+++' examples/code-diff.patch
command grep -E -c 'AKIA[0-9A-Z]{16}' examples/code-diff.patch examples/code-files.txt
command grep -E -n 'AKIA[0-9A-Z]{16}|BEGIN [A-Z ]*PRIVATE KEY' examples/code-secret.patch
```
Attendu : premier grep `4` (4 fichiers dans le patch) ; deuxième `0` pour chacun (exit 1 — fixtures review propres) ; troisième : 2 lignes matchées (AKIA ligne 9, PRIVATE KEY ligne 12 du fichier patch).

- [ ] **Step 5: Commit**

```bash
command git add examples/
command git commit -m "feat(fixtures): paquet code planté (5 défauts) + fixture scan secret"
```

---

### Task 3: Skill /devil-code (unitaire)

**Files:**
- Create: `skills/devil-code/SKILL.md`

**Interfaces:**
- Consumes: contrat Task 1, « Contrat de spawn agent » + VALIDATE_JQ code du header, agents existants `devil:devil-{gemini,glm,deepseek,opus}`.
- Produces: surface `/devil-code [target] [intent.md] [devil]`.

- [ ] **Step 1: Créer `skills/devil-code/SKILL.md`** (contenu complet)

````markdown
---
name: devil-code
description: "Review critique d'un changement de code par un avocat du diable au choix (Gemini par défaut, GLM, Deepseek, Opus) : PR GitHub, branche vs base, range de commits ou working tree. Score, verdict approve/rework/reject, issues ancrées file:ligne avec scénario d'échec. Triggers: /devil-code, 'review code', 'critique le code', 'devil sur le code'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Bash, Agent, AskUserQuestion, Edit
---

# /devil-code — Review critique d'un changement de code

Un devil externe juge un CHANGEMENT de code (jamais un stock) : bugs,
architecture, sécurité, performance, tests, maintenabilité — et alignement
sur l'intention si elle est fournie. Le run devil est hermétique : tout ce
qu'il voit est packagé par toi (l'orchestrateur) en inputs étiquetés.

## Syntaxe

```
/devil-code                       # auto : working tree sale, sinon branche vs base
/devil-code 123                   # PR GitHub (gh)
/devil-code main                  # branche courante vs main
/devil-code abc12..def34          # entre 2 commits
/devil-code HEAD~1                # le dernier commit
/devil-code 123 intent.md glm     # PR + doc d'intention + devil
```

Parsing des arguments : le dernier s'il vaut `gemini`, `glm`, `deepseek` ou
`opus` → devil (défaut `gemini`) ; un argument `*.md` existant → INTENT ;
le reste → target.

## Étape 0 — Chemins du plugin

Racine du plugin : deux niveaux au-dessus du « Base directory for this
skill » injecté ci-dessus. Résous en absolu :
- `SCHEMA_FILE` = `<racine>/scripts/devil-code-schema.json`
- `MISSION_FILE` = `<racine>/scripts/devil-code-mission.md`

Vérifie que les deux existent (Read). S'ils manquent, arrête et signale un
plugin corrompu.

## Étape 1 — Résolution du target

| Forme | Mode | Résolution du diff |
|---|---|---|
| nombre pur (`123`) | PR | `gh pr view 123 --json title,body,baseRefName,headRefName,state,additions,deletions,changedFiles` + `gh pr diff 123` |
| contient `..` (`a..b`) | range | `git diff a..b` |
| ref valide (`git rev-parse --verify`) | branche vs base | `git diff ref...HEAD` |
| *(rien)*, tree sale (`git status --porcelain` non vide) | working tree | `git diff HEAD` (staged + unstaged) |
| *(rien)*, tree propre | branche vs base auto | base = HEAD branch d'`origin` (fallback `main` puis `master` locale) → `git diff base...HEAD` |

Gardes et cas limites (STOP = arrêt propre avec le message, AUCUN appel
modèle) :
- Hors repo git → STOP.
- Ref/PR introuvable → STOP avec le message git/gh.
- Diff vide → STOP « rien à reviewer ».
- `gh` non installée en mode PR → STOP « gh CLI requis pour le mode PR ».
- Arg qui est un chemin existant (`src/index.ts`, `lib/`) → STOP « target =
  PR, range ou ref ; la review d'un fichier/dossier sans notion de
  changement est hors périmètre ».
- Diff ne contenant QUE des fichiers binaires → STOP « aucun contenu texte
  à reviewer ».

INTENT, par priorité : arg `*.md` → body de la PR (mode PR, écrit dans un
fichier temp) → absent. Pas d'auto-detect `.specs/`.

## Étape 2 — Packaging hermétique

`TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/devil-code-XXXXXX")`, trois fichiers :

- **`$TMP_DIR/diff.patch` (DIFF)** — le diff unifié brut, jamais tronqué.
  S'il dépasse 1 Mo → STOP et suggère de découper (range plus petit).
- **`$TMP_DIR/files.txt` (FILES)** — état final de chaque fichier texte
  modifié, chacun sous `=== FILE: chemin ===` ; fichier supprimé →
  `=== FILE: chemin (supprimé) ===` sans contenu ; binaires exclus et
  listés. Source selon le mode :

| Mode | Source FILES | Correction guidée (Étape 8) |
|---|---|---|
| working tree | `cat` du checkout | oui |
| branche vs base (tree propre) | `cat` du checkout (HEAD) | oui |
| range `a..b`, `b` = HEAD, tree propre | `cat` du checkout | oui |
| range `a..b`, `b` = HEAD, tree sale | `git show HEAD:chemin` | non — le working tree ≠ le diff reviewé |
| range historique (`b` ≠ HEAD) | `git show b:chemin` | non — rapport seul |
| PR checkoutée localement, tree propre | `cat` du checkout | oui |
| PR checkoutée, tree sale | `git show HEAD:chemin` | non — rapport seul |
| PR non checkoutée | **FILES omis** + badge « REVIEW SUR DIFF SEUL » | non — rapport seul |

  `git show b:chemin` en échec (rename compliqué) → fichier listé
  `=== FILE: chemin (indisponible au commit b) ===`, la review continue.
  Budget FILES : 200 Ko. Au-delà : tri des fichiers par lignes de diff
  décroissantes, garde jusqu'au budget, exclus LISTÉS en tête du paquet
  (`=== FILES EXCLUS (budget) : … ===`) et dans la confirmation.
- **`$TMP_DIR/intent.md` (INTENT)** — copie du doc d'intention ou body PR,
  si présent.

## Étape 3 — Scan pré-vol anti-fuite (AVANT tout envoi)

1. **Exclusions dures du paquet FILES** (glob, liste figée) : `.env*`,
   `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `id_ed25519*`,
   `credentials*`, `secrets*`, `*.keystore`. Fichier exclu = listé dans la
   confirmation ; ses hunks restent dans le DIFF, couverts par le scan
   ci-dessous.
2. **Scan de contenu** sur DIFF + FILES + INTENT (`command grep -E -n`,
   ajouter `-i` UNIQUEMENT pour la ligne « affectations ») :

| Cible | Pattern |
|---|---|
| Clés privées | `-----BEGIN [A-Z ]*PRIVATE KEY` |
| AWS access key | `AKIA[0-9A-Z]{16}` |
| GitHub tokens | `ghp_[A-Za-z0-9]{36}` et `github_pat_[A-Za-z0-9_]{22,}` |
| Clés style sk- | `sk-[A-Za-z0-9_-]{20,}` |
| Slack | `xox[baprs]-[A-Za-z0-9-]{10,}` |
| Google | `AIza[0-9A-Za-z_-]{35}` |
| JWT | `eyJ[A-Za-z0-9_-]{20,}\.eyJ` |
| Affectations (avec -i) | `(password|passwd|secret|token|api[_-]?key|auth)[[:space:]]*[:=][[:space:]]*['"][^'"]{8,}` |
| URL à credentials | `[a-z][a-z0-9+.-]*://[^/[:space:]:]+:[^@[:space:]]+@` |

3. Un hit → **STOP avant tout envoi** : affiche les lignes suspectes
   (fichier:ligne + extrait), Romain tranche :
   - **exclure** : régénère le DIFF avec `git diff … -- ':!chemin'` (mode
     PR : retire les hunks du fichier du diff téléchargé), retire le
     fichier de FILES, re-scanne le paquet réduit ;
   - **annuler** : fin de la skill, `trash "$TMP_DIR"` ;
   - **forcer** : continue, et la confirmation porte « scan : FORCÉ ».
   Biais assumé : le faux positif (un STOP à tort coûte une relecture, un
   faux négatif coûte une fuite irréversible).

## Étape 4 — Confirmation

Toujours confirmer avant de lancer :

> **Contexte détecté :**
> - Mode : <PR 123 / branche vs main / range a..b / working tree>
> - Fichiers : <N> (±<lignes>) · FILES <complet / tronqué : n exclus / omis>
> - Intent : <chemin / body PR / absent>
> - Scan : <clean / forcé> · Correction guidée : <disponible / rapport seul>
> - Devil : <devil> (<modèle>)
>
> Je lance la review ?

Attends la confirmation de Romain (« oui », « go », « lance »).

## Étape 5 — Lancer le sous-agent

Spawn `devil:devil-<devil>` ; si ce type est introuvable (plugin non
chargé), retente `devil-<devil>` sans préfixe. INPUTS : `DIFF:` toujours ;
`FILES:` et `INTENT:` seulement si les fichiers existent :

```
Agent(
  subagent_type: "devil:devil-<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.score|type==\"number\" and .>=0 and .<=100) and (.verdict|IN(\"approve\",\"rework\",\"reject\")) and (.criteria|has(\"correctness\") and has(\"architecture\") and has(\"security\") and has(\"performance\") and has(\"tests\") and has(\"maintainability\")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type==\"object\" and has(\"score\") and has(\"comment\") and (.score|type==\"number\" and .>=0 and .<=100))) and (.issues|type==\"array\" and all(has(\"severity\") and has(\"category\") and has(\"file\") and has(\"description\") and has(\"failure_scenario\") and has(\"suggestion\") and (.severity|IN(\"critical\",\"high\",\"medium\",\"low\")) and (.category|IN(\"correctness\",\"architecture\",\"security\",\"performance\",\"tests\",\"maintainability\",\"intent\"))))\nINPUTS:\nDIFF:<abs diff>\nFILES:<abs files>\nINTENT:<abs intent>\n\nExécute la procédure de transport."
)
```

Annonce avant le spawn : « **Review en cours…** <devil> analyse le
changement (jusqu'à 9 min). »

## Étape 6 — Ancrage puis rapport

Le retour est UNE ligne JSON : `{devil, model, status, review|error+detail}`.

### Si `status: "error"`
```
══════ REVIEW ÉCHOUÉE ══════

Devil : <devil> (<model>)
Erreur : <error> — <detail>

→ Relance (/devil-code <target> <devil>), autre devil, ou review manuelle.
```

### Si `status: "ok"` — ancrage d'abord

Extrais du DIFF les plages de lignes modifiées PAR FICHIER (en-têtes de
hunks `@@ -a,b +c,d @@`, côté état final `+c,d`). Pour chaque issue :
1. fichier hors du périmètre du diff → **DÉCLASSÉE** ;
2. ligne hors des plages modifiées, tolérance ±3 lignes → **DÉCLASSÉE** ;
   fichier supprimé → ancrage au fichier seul.

Les DÉCLASSÉES vont dans une section « Non ancrées » sous le tableau
(jamais supprimées en silence) et sont exclues de la correction guidée.

### Rapport

```
══════ CODE REVIEW ══════
[REVIEW SUR DIFF SEUL — contexte réduit]        ← si FILES omis

Devil : <devil> (<model>) · Cible : <mode/cible>
Score : <score>/100  [APPROVE ✓ | REWORK ✗ | REJECT ☠]

Critères :
  Correctness      <n>/100  <commentaire court>
  Architecture     <n>/100  <commentaire court>
  Security         <n>/100  <commentaire court>
  Performance      <n>/100  <commentaire court>
  Tests            <n>/100  <commentaire court>
  Maintainability  <n>/100  <commentaire court>

Résumé : <summary>
```

Issues (si non vide), tableau trié par sévérité (critical > high > medium
> low) :

```
| Sév | Cat | Fichier | Problème | Scénario d'échec | Suggestion |
```

Puis la section « Non ancrées » le cas échéant.

## Étape 7 — Next steps selon le verdict

### approve (score ≥ 80)
> **Verdict : APPROVE** — Le changement est solide. [issues mineures s'il y en a]
> → Prêt à commit/merge ?

### rework (50-79)
> **Verdict : REWORK** — <n> issues à adresser.
> 1. **Corriger** — correction guidée (si le mode l'autorise, Étape 8)
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-review** — je corrige puis je relance le même devil

### reject (< 50)
> **Verdict : REJECT** — le changement est structurellement mauvais :
> réécrire coûte moins cher que corriger. Pas de correction incrémentale :
> on rouvre la conception.

Attends la décision de Romain.

## Étape 8 — Correction guidée (si demandée ET si le mode l'autorise)

Modes autorisés : voir la colonne « Correction guidée » de la table de
l'Étape 2. Sinon : rapport seul, le proposer explicitement.

Séquence par issue, sévérité décroissante (`low` ignorées sauf demande,
non-ancrées exclues) :
1. présente l'issue (fichier:ligne, problème, scénario, suggestion) ;
2. Romain tranche : **appliquer** / **ignorer** / **stop** (arrêt de la
   passe, récap immédiat) ;
3. si appliquer : relis le fichier concerné, puis Edit ;
4. issue suivante ; à la fin, récap une ligne par issue (appliquée /
   ignorée), puis proposer : **re-review** (max 2, même devil, mêmes
   target/intent) ou s'arrêter là.

## Règles

- Le devil ne modifie JAMAIS rien : c'est toi qui édites, avec les
  décisions de Romain uniquement.
- Les issues `low` ne sont PAS corrigées sauf demande explicite.
- Maximum 2 cycles de re-review ; après 2 rework consécutifs, Romain
  tranche.
- `trash "$TMP_DIR"` en fin de run (succès comme échec).
- Jamais de secrets dans un paquet envoyé : le scan de l'Étape 3 est
  obligatoire, jamais sauté.
````

- [ ] **Step 2: Vérifier**

```bash
command grep -n 'name: devil-code' skills/devil-code/SKILL.md
command grep -c '~/.claude' skills/devil-code/SKILL.md
command grep -c 'BRAINSTORMING:' skills/devil-code/SKILL.md
```
Attendu : ligne 2 matchée ; deuxième et troisième grep `0` (exit 1 — pas de chemin en dur, pas de label d'un autre exercice).

- [ ] **Step 3: Commit**

```bash
command git add skills/devil-code/
command git commit -m "feat(skills): /devil-code — review critique d'un changement de code"
```

---

### Task 4: Skill /devil-code-swarm (tribunal)

**Files:**
- Create: `skills/devil-code-swarm/SKILL.md`

**Interfaces:**
- Consumes: contrat Task 1, skill Task 3 (étapes 0 à 4 partagées par référence), agents `devil:devil-{gemini,glm,deepseek}` (opus EXCLU, décision actée).
- Produces: surface `/devil-code-swarm [target] [intent.md]`.

- [ ] **Step 1: Créer `skills/devil-code-swarm/SKILL.md`** (contenu complet ; le VALIDATE_JQ du prompt est BYTE-IDENTIQUE à celui de la Task 3)

````markdown
---
name: devil-code-swarm
description: "Tribunal du code : Gemini + GLM + Deepseek reviewent le même changement en parallèle, consolidation par problème de fond, verdict VALABLE / MODIFICATIONS REQUISES / JETABLE avec garde-fou sécurité. Triggers: /devil-code-swarm, 'swarm sur le code', 'tribunal du code'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Bash, Agent, AskUserQuestion, Edit
---

# /devil-code-swarm — Le tribunal du code

Les 3 devils (gemini + glm + deepseek — opus n'est pas convoqué au
tribunal, choix acté) jugent le MÊME changement EN PARALLÈLE. Toi
(l'orchestrateur) tu consolides : ancrage par voix, convergences, verdict
avec garde-fou sécurité.

## Étapes 0 à 4 — identiques à /devil-code

Chemins plugin, résolution du target (mêmes formes, gardes et cas
limites), packaging (mêmes règles DIFF/FILES/INTENT, un SEUL TMP_DIR
partagé en lecture par les 3 spawns), scan pré-vol (obligatoire, jamais
sauté) : déroule les Étapes 0 à 4 de `skills/devil-code/SKILL.md`, à une
différence près — l'annonce :

> **Tribunal du code :** gemini + glm + deepseek en parallèle (jusqu'à
> 9 min). Cible : <mode/cible>. Je lance ?

## Étape 5 — Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message. Prompt
IDENTIQUE pour les trois, seul le subagent_type change :
`devil:devil-gemini`, `devil:devil-glm`, `devil:devil-deepseek` (fallback
sans préfixe si un type est introuvable) :

```
Agent(
  subagent_type: "devil:devil-<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.score|type==\"number\" and .>=0 and .<=100) and (.verdict|IN(\"approve\",\"rework\",\"reject\")) and (.criteria|has(\"correctness\") and has(\"architecture\") and has(\"security\") and has(\"performance\") and has(\"tests\") and has(\"maintainability\")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type==\"object\" and has(\"score\") and has(\"comment\") and (.score|type==\"number\" and .>=0 and .<=100))) and (.issues|type==\"array\" and all(has(\"severity\") and has(\"category\") and has(\"file\") and has(\"description\") and has(\"failure_scenario\") and has(\"suggestion\") and (.severity|IN(\"critical\",\"high\",\"medium\",\"low\")) and (.category|IN(\"correctness\",\"architecture\",\"security\",\"performance\",\"tests\",\"maintainability\",\"intent\"))))\nINPUTS:\nDIFF:<abs diff>\nFILES:<abs files>\nINTENT:<abs intent>\n\nExécute la procédure de transport."
)
```

(`FILES:` et `INTENT:` seulement si les fichiers existent, comme en
unitaire.)

## Étape 6 — Collecte et quorum

- 3 voix `ok` → synthèse pleine.
- 2 voix `ok` → continue, le rapport OUVRE sur la voix absente :
  « ⚠ <devil> muet (<error> — <detail court>). Synthèse sur 2 voix. »
- ≤ 1 voix `ok` → PAS de verdict. Rapport d'échec, proposer : relancer le
  swarm, ou l'unitaire (/devil-code <target> <devil>).

Un retour qui n'est pas une enveloppe JSON valide compte comme voix
absente (ne JAMAIS interpréter un texte d'erreur comme une review).

## Étape 7 — Ancrage par voix, puis consolidation

D'abord l'ancrage de /devil-code Étape 6, appliqué PAR VOIX (les
DÉCLASSÉES d'une voix sortent de sa contribution avant consolidation, et
restent listées en « Non ancrées » avec leur devil).

Puis consolidation par PROBLÈME DE FOND — ton travail LLM natif, AUCUN
algorithme, embedding ni seuil à coder. Heuristiques d'aide au jugement :
- candidats à l'équivalence : même fichier ET plages de lignes
  chevauchantes ou voisines (±10) ET même problème de fond (la catégorie
  aide mais ne décide pas — un même bug peut être classé correctness par
  l'un, security par l'autre) ;
- deux issues au même endroit peuvent rester deux problèmes distincts ;
  en cas de doute, NE PAS fusionner ;
- badge de convergence 3/3, 2/3, 1/3 (sur les voix exprimées : à 2 voix,
  2/2 et 1/2) ; sévérité du groupe = la plus haute ; suggestion la plus
  actionnable conservée.
- Tri : convergence décroissante, puis sévérité max du groupe.

## Étape 8 — Verdict

| Situation (voix exprimées) | Verdict |
|---|---|
| ≥ 2 approve ET 0 reject | **VALABLE** |
| ≥ 2 reject | **JETABLE** |
| tout le reste (dont split approve/rework/reject) | **MODIFICATIONS REQUISES** |

**Garde-fou sécurité** : ≥ 1 issue `critical` de catégorie `security`
ANCRÉE portée par au moins un devil → le verdict ne peut pas être
VALABLE : un VALABLE issu de la table est ramené à MODIFICATIONS
REQUISES avec mention explicite du garde-fou ; un JETABLE reste JETABLE
(le garde-fou ne remonte jamais un verdict).

Pas de score consolidé : grille des scores par devil + moyenne indicative.

## Étape 9 — Rapport

```
══════ TRIBUNAL DU CODE ══════

[⚠ voix absente le cas échéant]
[REVIEW SUR DIFF SEUL — contexte réduit]        ← si FILES omis

Verdict : [VALABLE ✓ | MODIFICATIONS REQUISES ⚠ | JETABLE ☠]
[— garde-fou sécurité appliqué, le cas échéant]
<2-3 phrases d'argumentation à partir des verdicts et convergences>

Scores :
  Critère          gemini  glm  deepseek  (moyenne)
  Correctness        <n>   <n>    <n>       <n>
  Architecture       <n>   <n>    <n>       <n>
  Security           <n>   <n>    <n>       <n>
  Performance        <n>   <n>    <n>       <n>
  Tests              <n>   <n>    <n>       <n>
  Maintainability    <n>   <n>    <n>       <n>
  GLOBAL             <n>   <n>    <n>       <n>

Verdicts : gemini <verdict> · glm <verdict> · deepseek <verdict>
```

Issues consolidées :

```
| Conv | Sév | Cat | Fichier | Problème | Suggestion | Devils |
```

Puis « Non ancrées » (avec devil d'origine) et « Voix dissonante » si
applicable (reject isolé, écart de score > 30 sur un critère : qui, sur
quoi, son argument principal — la majorité ne l'écrase JAMAIS
silencieusement).

## Étape 10 — Next steps selon le verdict

### VALABLE
> Le tribunal valide. [Mentionner les issues 2/3+ restantes]
> → Prêt à commit/merge ?

### MODIFICATIONS REQUISES
> 1. **Corriger** — correction guidée (si le mode l'autorise) : issues
>    convergentes (3/3, 2/3) d'abord, puis critical/high isolées
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-swarm** — je corrige puis je reconvoque le tribunal (1 max)

### JETABLE
> Le tribunal enterre le changement : <les 1-2 raisons de fond
> convergentes>. Pas de correction incrémentale : on rouvre la conception.

## Règles

- Correction guidée : mêmes règles et même séquence que /devil-code
  Étape 8 (modes autorisés, low ignorées, non-ancrées exclues, le devil
  ne modifie jamais rien).
- Maximum 1 re-swarm ; ensuite Romain tranche.
- opus ne siège pas au tribunal (dispo en unitaire : /devil-code opus).
- `trash "$TMP_DIR"` en fin de run.
- Coût : 3 modèles en parallèle ≈ la durée du plus lent. C'est un gate de
  commit/merge, pas un lint : pas sur un diff de deux lignes.
````

- [ ] **Step 2: Vérifier — dont l'identité byte-à-byte des VALIDATE_JQ jumeaux**

```bash
command grep -n 'name: devil-code-swarm' skills/devil-code-swarm/SKILL.md
command grep -c '~/.claude' skills/devil-code-swarm/SKILL.md
command grep -h 'VALIDATE_JQ=' skills/devil-code/SKILL.md skills/devil-code-swarm/SKILL.md | sort -u | wc -l
```
Attendu : ligne 2 matchée ; grep `~/.claude` = `0` (exit 1) ; dernier compte = `1` (les lignes VALIDATE_JQ complètes des deux skills sont byte-identiques — une seule variante après dédoublonnage).

- [ ] **Step 3: Commit**

```bash
command git add skills/devil-code-swarm/
command git commit -m "feat(skills): /devil-code-swarm — tribunal du code avec garde-fou sécurité"
```

---

### Task 5: Manifest v0.3.0, README, _memory_

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `README.md`
- Modify: `_memory_/architecture.md`, `_memory_/key-files.md`, `_memory_/patterns.md`

**Interfaces:**
- Consumes: tout ce qui précède.
- Produces: plugin v0.3.0 auto-cohérent, mémoire projet à jour, prêt pour les smokes.

- [ ] **Step 1: `.claude-plugin/plugin.json` — contenu complet**

```json
{
  "$schema": "https://www.schemastore.org/claude-code-plugin-manifest.json",
  "name": "devil",
  "description": "Avocats du diable : reviews critiques de specs (score, verdict, issues), interrogatoire socratique de brainstormings (les questions jamais posées) et review de changements de code (PR, branche, range, working tree), par Gemini, GLM, Deepseek et Opus, unitaires ou en swarm.",
  "version": "0.3.0",
  "author": {
    "name": "Romain Ecarnot",
    "url": "https://github.com/eRom"
  },
  "repository": "https://github.com/eRom/erom-agence-devil",
  "license": "MIT",
  "keywords": ["review", "specs", "brainstorming", "code-review", "devils-advocate", "socratic", "gemini", "glm", "deepseek", "opus", "swarm"],
  "skills": "./skills/"
}
```
(Toujours PAS de clé `agents` : rejetée par le schéma, auto-découverte.)

- [ ] **Step 2: `README.md` — trois remplacements**

Remplacement 1 — l'intro (les deux angles → trois) ; ancien :
```
- **devil-spec** : ils jugent une spec technique contre son brainstorm
  d'origine (dérives, manques, incohérences), score et verdict à la clé.
- **devil-brain** : ils interrogent un brainstorming seul et rendent les
  questions les plus dangereuses jamais posées (angles morts, pans oubliés),
  sans score ni verdict.
```
Nouveau :
```
- **devil-spec** : ils jugent une spec technique contre son brainstorm
  d'origine (dérives, manques, incohérences), score et verdict à la clé.
- **devil-brain** : ils interrogent un brainstorming seul et rendent les
  questions les plus dangereuses jamais posées (angles morts, pans oubliés),
  sans score ni verdict.
- **devil-code** : ils jugent un changement de code (PR, branche, range de
  commits ou working tree) — bugs, architecture, sécurité, performance,
  tests, maintenabilité — avec scan anti-fuite de secrets avant tout envoi.
```

Remplacement 2 — la table des devils (ajout d'opus, absent du README
depuis la 0.2.1) ; ancien :
```
| Devil | Modèle | Transport |
|---|---|---|
| gemini | Gemini 3.5 Flash (High) | Antigravity CLI (agy) |
| glm | glm-5.2:cloud | claude CLI → ollama cloud |
| deepseek | deepseek-v4-pro:cloud | claude CLI → ollama cloud |
```
Nouveau :
```
| Devil | Modèle | Transport | Swarms |
|---|---|---|---|
| gemini | Gemini 3.5 Flash (High) | Antigravity CLI (agy) | oui |
| glm | glm-5.2:cloud | claude CLI → ollama cloud | oui |
| deepseek | deepseek-v4-pro:cloud | claude CLI → ollama cloud | oui |
| opus | Opus 4.8 xHigh | claude CLI | non (unitaire seulement) |
```

Remplacement 3 — le bloc Usage et les paragraphes ; ancien :
```
/devil-spec                          # specs vs brainstorm, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
/devil-brain                         # questions socratiques sur un brainstorming
/devil-brain brainstorming.md deepseek
/devil-brain-swarm                   # les 3 voix, consolidation par convergence
```
Nouveau :
```
/devil-spec                          # specs vs brainstorm, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
/devil-brain                         # questions socratiques sur un brainstorming
/devil-brain brainstorming.md deepseek
/devil-brain-swarm                   # les 3 voix, consolidation par convergence
/devil-code                          # review du changement courant (auto)
/devil-code 123 glm                  # review d'une PR GitHub
/devil-code main intent.md           # branche vs main, avec doc d'intention
/devil-code-swarm HEAD~1             # tribunal sur le dernier commit
```
Et APRÈS le paragraphe `**devil-brain** — …`, ajouter :
```

**devil-code** — entrée : un changement (PR via gh, branche vs base, range
de commits, working tree), packagé en DIFF + fichiers modifiés + intention
optionnelle. Scan anti-fuite de secrets AVANT tout envoi (STOP sur hit).
Sortie par devil : JSON strict (score 0-100, verdict approve/rework/reject,
6 critères code, issues ancrées file:ligne avec scénario d'échec
obligatoire). Le swarm (gemini + glm + deepseek, opus exclu) consolide par
problème de fond : VALABLE, MODIFICATIONS REQUISES ou JETABLE, avec
garde-fou sécurité (une critical security ancrée interdit VALABLE).
```

- [ ] **Step 3: `_memory_/architecture.md` — trois remplacements**

Remplacement 1 ; ancien :
```
> MàJ : 2026-07-18 (v0.2.0)

**Type** : Plugin Claude Code `devil` (v0.2.0), distribué par `erom-marketplace`.
```
Nouveau :
```
> MàJ : 2026-07-18 (v0.3.0)

**Type** : Plugin Claude Code `devil` (v0.3.0), distribué par `erom-marketplace`.
```

Remplacement 2 ; ancien :
```
- **brain** : interrogatoire socratique d'un brainstorming seul — les 5
  questions les plus dangereuses jamais posées, SANS score ni verdict ;
  0 question = prêt à spécifier (signal faible, limite actée).
```
Nouveau :
```
- **brain** : interrogatoire socratique d'un brainstorming seul — les 5
  questions les plus dangereuses jamais posées, SANS score ni verdict ;
  0 question = prêt à spécifier (signal faible, limite actée).
- **code** : review d'un CHANGEMENT (PR/branche/range/working tree) packagé
  hermétiquement (DIFF + FILES + INTENT opt.), scan anti-fuite pré-vol,
  ancrage file:ligne vérifié au retour, garde-fou sécurité en swarm (opus
  exclu du tribunal, dispo en unitaire).
```

Remplacement 3 — l'arborescence ; ancien :
```
.claude-plugin/plugin.json      manifest 0.2.0 (PAS de clé agents)
agents/devil-{gemini,glm,deepseek}.md      transport pur
skills/devil-spec{,-swarm}/     exercice spec (2 inputs BRAINSTORMING+SPECS)
skills/devil-brain{,-swarm}/    exercice brain (1 input BRAINSTORMING)
scripts/devil-spec-{mission.md,schema.json}
scripts/devil-brain-{mission.md,schema.json}
examples/                       fixtures veilleur (6 défauts plantés)
.specs/plugin-devil{,-brain}/   design v0.1.0 et v0.2.0 (brainstorm+archi+plan)
```
Nouveau :
```
.claude-plugin/plugin.json      manifest 0.3.0 (PAS de clé agents)
agents/devil-{gemini,glm,deepseek,opus}.md transport pur (opus hors swarms)
skills/devil-spec{,-swarm}/     exercice spec (2 inputs BRAINSTORMING+SPECS)
skills/devil-brain{,-swarm}/    exercice brain (1 input BRAINSTORMING)
skills/devil-code{,-swarm}/     exercice code (DIFF + FILES/INTENT opt.)
scripts/devil-{spec,brain,code}-{mission.md,schema.json}
examples/                       fixtures veilleur (6 défauts) + code (5 défauts + secret)
.specs/plugin-devil{,-brain,-code}/  designs v0.1.0, v0.2.0, v0.3.0
```

- [ ] **Step 4: `_memory_/key-files.md` — deux ajouts**

Après le bloc `- devil-brain-mission.md + …` de la section « Contrats par
exercice », ajouter :
```
- `devil-code-mission.md` + `devil-code-schema.json` — exercice code :
  6 critères code scorés {score,comment}, verdict approve|rework|reject,
  issues {severity, category (6+intent), file "chemin:ligne",
  failure_scenario obligatoire, suggestion}. VALIDATE_JQ borne scores ET
  critères (testée jq 1.8.1).
```
Après le bloc `- devil-brain-swarm/SKILL.md — …` de la section « Skills »,
ajouter :
```
- `devil-code/SKILL.md` — unitaire code ; résolution target (PR/range/
  ref/auto), packaging TMP_DIR (DIFF jamais tronqué, FILES budget 200 Ko,
  INTENT opt.), scan pré-vol AVANT envoi (STOP sur hit), ancrage
  file:ligne ±3 sur les hunks, correction guidée selon mode (table).
- `devil-code-swarm/SKILL.md` — tribunal code (opus exclu) ; ancrage par
  voix, consolidation problème de fond, verdict table + garde-fou
  sécurité (critical security ancrée → jamais VALABLE), tri convergence
  puis sévérité.
```
Et dans la section « Fixtures & specs », après la ligne `examples/…
veilleur`, ajouter :
```
- `examples/code-diff.patch` + `code-files.txt` — paquet code planté
  (injection db.ts:17, null deref auth.ts:9, N+1 report.ts:8, dup
  maskName, test sans assertion). `code-secret.patch` — oracle du scan
  pré-vol (AKIA + PRIVATE KEY, exemples doc AWS).
```

- [ ] **Step 5: `_memory_/patterns.md` — deux ajouts**

Dans la section « Synthèses swarm », après la ligne `brain : …`, ajouter :
```
- code : verdict = table spec + GARDE-FOU (critical security ancrée →
  jamais VALABLE, ne remonte jamais un JETABLE). Ancrage PAR VOIX avant
  consolidation. Tri convergence puis sévérité.
```
En fin de fichier, ajouter la section :
```

## Exercice code (v0.3.0)
- Le devil juge un CHANGEMENT, jamais un stock : arg chemin → STOP.
- Packaging orchestrateur : DIFF jamais tronqué (> 1 Mo → refus), FILES
  budget 200 Ko (tri par lignes de diff, exclus listés), INTENT jamais
  auto-détecté (.specs interdit — arg explicite ou body PR).
- Scan pré-vol AVANT tout envoi : globs d'exclusion + 9 regex figées
  (biais faux positif assumé). Hit → STOP, Romain tranche
  exclure/annuler/forcer.
- Ancrage au retour : file:ligne vs plages de hunks ±3 ; hors périmètre →
  DÉCLASSÉE visible (jamais supprimée), exclue de la correction et du
  garde-fou.
- Correction guidée UNIQUEMENT si le diff reviewé = working tree actuel
  (tree propre requis hors mode working tree).
```

- [ ] **Step 6: Inventaire plugin**

```bash
claude --plugin-dir /Users/recarnot/dev/erom-agence-devil plugin details devil
```
Attendu : version 0.3.0, 6 skills (devil-spec, devil-spec-swarm, devil-brain, devil-brain-swarm, devil-code, devil-code-swarm), 4 agents (devil-gemini, devil-glm, devil-deepseek, devil-opus).

- [ ] **Step 7: Grep final anti-fuites de périmètre (`.specs/` et `_memory_` exclus : historique)**

```bash
command grep -rn 'BRAINSTORMING:\|SPECS:' skills/devil-code/ skills/devil-code-swarm/
command grep -rn '~/.claude' skills/ scripts/
```
Attendu : aucune sortie pour les deux (exit 1).

- [ ] **Step 8: Commit**

```bash
command git add .claude-plugin/plugin.json README.md _memory_/
command git commit -m "chore(plugin): v0.3.0 — manifest, README, mémoire projet (exercice code)"
```

---

### Task 6: Smokes live (scan, code, erreur)

Aucun fichier créé (vérifications). Chaque spawn d'agent de test suit la leçon v0.2.0 : le prompt exige l'écriture de l'enveloppe dans un fichier scratchpad EN PLUS du retour (canal teammate intermittent).

- [ ] **Step 1: Oracle du scan pré-vol sur la fixture secret (déterministe, sans modèle)**

```bash
command grep -E -n -e '-----BEGIN [A-Z ]*PRIVATE KEY' -e 'AKIA[0-9A-Z]{16}' -e 'ghp_[A-Za-z0-9]{36}' -e 'github_pat_[A-Za-z0-9_]{22,}' -e 'sk-[A-Za-z0-9_-]{20,}' -e 'xox[baprs]-[A-Za-z0-9-]{10,}' -e 'AIza[0-9A-Za-z_-]{35}' -e 'eyJ[A-Za-z0-9_-]{20,}\.eyJ' -e '[a-z][a-z0-9+.-]*://[^/[:space:]:]+:[^@[:space:]]+@' examples/code-secret.patch
command grep -E -n -i -e "(password|passwd|secret|token|api[_-]?key|auth)[[:space:]]*[:=][[:space:]]*['\"][^'\"]{8,}" examples/code-secret.patch
command grep -E -c -e '-----BEGIN [A-Z ]*PRIVATE KEY' -e 'AKIA[0-9A-Z]{16}' examples/code-diff.patch examples/code-files.txt
```
Attendu : premier grep ≥ 2 lignes (AKIA + PRIVATE KEY) ; deuxième : aucune ligne, exit 1 (`secretAccessKey` ne matche PAS le pattern affectations — « secret » n'y est pas suivi de `:=` ; c'est le pattern AKIA qui couvre ce fichier) ; troisième : `0` pour chacun (exit 1 — les fixtures review sont propres au scan).

- [ ] **Step 2: Smoke code unitaire (glm, le moins cher) sur le paquet fixture**

Spawn UN agent `general-purpose` avec ce prompt (le VALIDATE_JQ est celui du header, en une ligne) :
```
Lis /Users/recarnot/dev/erom-agence-devil/agents/devil-glm.md et exécute
EXACTEMENT sa procédure (corps, pas le frontmatter) avec :
MISSION_FILE=/Users/recarnot/dev/erom-agence-devil/scripts/devil-code-mission.md
SCHEMA_FILE=/Users/recarnot/dev/erom-agence-devil/scripts/devil-code-schema.json
VALIDATE_JQ=<la ligne VALIDATE_JQ code du header, verbatim>
INPUTS:
DIFF:/Users/recarnot/dev/erom-agence-devil/examples/code-diff.patch
FILES:/Users/recarnot/dev/erom-agence-devil/examples/code-files.txt

Écris l'enveloppe finale dans <scratchpad>/smoke-code-glm.json ET
retourne-la. Uniquement le JSON, aucun texte autour.
```
CRITÈRES DE PASSAGE (tous requis) :
1. Enveloppe `status:"ok"`, VALIDATE_JQ passée (sinon l'agent aurait rendu SCHEMA_INVALID).
2. `verdict` ∈ {rework, reject} (5 défauts plantés doivent tirer le score sous 80).
3. Au moins 4 des 5 défauts plantés retrouvés ou approchés : injection SQL db.ts (attendu critical/security), null deref auth.ts, N+1 report.ts, duplication maskName, test sans assertion.
4. TOUTES les issues ancrées : chaque `file` ∈ {src/db.ts, src/auth.ts, src/report.ts, tests/auth.test.ts} et chaque ligne dans les plages des hunks ±3 (db.ts : lignes 15-20 du patch ; fichiers neufs : toute ligne 1..N).
5. `failure_scenario` non vide et concret sur chaque issue ; ZÉRO issue de style/naming pur.
6. L'injection db.ts porte `severity:"critical"` et `category:"security"` — c'est l'oracle du garde-fou swarm (avec cette issue ancrée, un verdict VALABLE serait impossible au tribunal).

- [ ] **Step 3: Chemin d'erreur (script direct, modèle bidon — déterministe)**

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

- [ ] **Step 4: Si un smoke échoue** — diagnostiquer avec l'enveloppe error (jamais stderr seul), corriger le fichier concerné (mission, schéma ou skill), committer le fix (`fix(…): …`), relancer le smoke concerné. Ne JAMAIS passer à la Task 7 avec un smoke rouge. Si le smoke code rate ≥ 2 défauts plantés à deux runs consécutifs, STOP : c'est un problème de mission (anti-bruit trop agressif ou packaging illisible), à remonter à Romain avant tout rework.

---

### Task 7: Dogfood méta — /devil-code sur sa propre branche

Vérification finale en conditions réelles, ET premier usage réel de la skill : le juge de code jugé par ses collègues sur son propre chantier.

- [ ] **Step 1: Dérouler `skills/devil-code/SKILL.md` à la main** (les types `devil:devil-*` du plugin installé pointent la 0.2.1 — fallback prescrit : spawn `general-purpose` « lis agents/devil-glm.md et exécute », comme Task 6). Target : `main...HEAD` de la branche d'implémentation (mode branche vs base). Étapes 1 à 4 de la skill déroulées TELLES QU'ÉCRITES : résolution, packaging (FILES depuis le checkout), scan pré-vol (les SKILL.md contiennent les patterns de scan en clair : des hits d'auto-détection sur ces fichiers sont attendus → option « forcer », c'est le cas nominal documenté), confirmation à Romain.

- [ ] **Step 2: Ancrage + rapport selon les Étapes 6-7 de la skill.** Présenter le rapport à Romain. Attendu réaliste : verdict `approve` ou `rework` avec issues actionnables (le chantier vient d'être reviewé par tribunal de specs, un `reject` signalerait un problème sérieux — STOP et diagnostic avant merge).

- [ ] **Step 3: Corrections éventuelles** selon la décision de Romain (séquence de l'Étape 8 de la skill), commits `fix(…)`. Le dogfood doit être VERT (approve, ou rework corrigé/assumé par Romain) avant la Task 8.

---

### Task 8: Livraison (après merge sur main — hors branche)

PRÉALABLE : la branche est terminée via superpowers:finishing-a-development-branch (Tasks 6-7 vertes), Romain a choisi le merge. NE PAS pousser sans son accord explicite.

- [ ] **Step 1: Marketplace** — Read `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json`, bump `metadata.version` de +0.1 (valeur courante lue dans le fichier, ne pas présumer — l'entrée `devil` pointe l'URL GitHub, rien d'autre à changer). Commit + push (HTTPS déjà configuré).

- [ ] **Step 2: Push du repo devil**

```bash
command git push origin main
```

- [ ] **Step 3: Réinstallation (uninstall PUIS install — gotcha mémoire ; attendre ~10 s après les push)**

```bash
claude plugin marketplace update erom-marketplace
claude plugin uninstall devil@erom-marketplace
claude plugin install devil@erom-marketplace
claude plugin details devil
```
Attendu : `devil@erom-marketplace v0.3.0`, scope user, enabled, 6 skills + 4 agents.

- [ ] **Step 4: Rappeler à Romain** que les nouvelles skills ne seront visibles qu'après redémarrage des sessions Claude Code ouvertes, et que l'ancienne skill perso `~/.claude/skills/devil-code` a déjà été trashée (collision de nom réglée le 2026-07-18).
