# devil review v0.7.0 - Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ajouter la porte de merge `/erom-devil:review` + `/erom-devil:review-swarm` au plugin erom-devil : porte déterministe, chasse à l'intention, grille de stack, critique devils, vérification contradictoire par l'orchestrateur, verdict GO/NO-GO, rapport persistant. Agents transport inchangés.

**Architecture:** 4e exercice du contrat v0.2.0 : 1 mission + 1 schéma + 2 skills, zéro modification des 5 agents. Les skills review référencent les étapes durcies de `skills/code/SKILL.md` (résolution du target, packaging, scan anti-fuite, ancrage) au lieu de les dupliquer, et ajoutent : porte déterministe, chasse à l'intention, input STACK, passe de vérification Claude, verdict GO/NO-GO, rapport dans `docs/reviews/`. Référence : `.specs/plugin-devil-review/{brainstorming,architecture-technique}.md` (commit b0dd7cc, version corrigée 0.7.0 ensuite).

**Tech Stack:** Markdown (skills plugin Claude Code), bash + `git` + `gh` + `jq` + `grep -E`, agents transport existants (`agy` Gemini, `claude -p` vers ollama cloud GLM/Deepseek/Kimi, `claude` Opus), `trash`.

**Spec:** `.specs/plugin-devil-review/architecture-technique.md` (+ `brainstorming.md` pour les décisions produit). Les exécutants lisent la spec ET ce plan.

## Global Constraints

- Jamais `rm` / `rmdir` : suppression via `trash` uniquement.
- Timeout Bash explicite **540000** ms + **1 retry** sur tout appel modèle (smokes des Tasks 6-8).
- Jamais `ollama pull`, jamais de préflight `/api/tags` (modèles `:cloud` non listés, normal et validé).
- Les 5 agents (`plugin/agents/{gemini,glm,deepseek,opus,kimi}.md`) sont INTOUCHÉS : toute tâche qui voudrait les modifier est hors périmètre.
- Surfaces `spec`, `brain` et `code` : INCHANGÉES. Aucun fichier de ces exercices n'est touché (`skills/code/SKILL.md` est RÉFÉRENCÉ par review, jamais modifié).
- Enums JSON en anglais, contenus rédigés en français.
- Zéro chemin `~/.claude/` en dur dans `plugin/skills/` et `plugin/scripts/`.
- VALIDATE_JQ : BYTE-IDENTIQUE dans les 4 skills code, code-swarm, review, review-swarm (une seule variante après `sort -u`). Ligne re-testée sur jq le 2026-08-18 : échantillon conforme = pass, issue sans `failure_scenario` = fail.
- **AUCUN tiret cadratin (U+2014) dans les fichiers rédigés** : le hook guard-emdash bloque le Write. Les fichiers hérités (`skills/code/SKILL.md`, missions existantes) en contiennent : ne jamais les copier-coller tels quels, réécrire avec « : », « - » ou reformuler. Les contenus fournis dans ce plan sont déjà propres : les transcrire tels quels. Dans les commandes de vérification, le caractère s'obtient par `"$(printf '\xe2\x80\x94')"`, jamais en littéral.
- Manifest : PAS de clé `agents` (rejetée par le schéma, auto-découverte) ; `--plugin-dir` est un flag GLOBAL (avant la sous-commande `plugin`).
- MàJ plugin installé = `uninstall` PUIS `install` (gotcha mémoire).
- Sorties de commande pilotant une décision : préfixer `command` (hook rtk réécrit git/grep/ls).
- Greps de non-présence : exit 1 = SUCCÈS attendu (0 occurrence), jamais un échec de tâche.
- Suffixe `[1m]` des modèles ollama : toujours QUOTÉ en bash (zsh glob nomatch sinon) ; ne concerne que les smokes.
- Spawn d'un devil : `subagent_type: "erom-devil:<devil>"`, fallback `<devil>` sans préfixe si type introuvable.
- Un seul committer à la fois : les commits sont faits par l'orchestrateur ou sérialisés, jamais deux subagents en parallèle sur l'index.
- Commits : trailers `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>` + `Claude-Session: https://claude.ai/code/session_01T6fXA8PWXeXpTMMmY2Q3fV` sur chaque commit.
- Tous les chemins de fichiers du plan sont relatifs à la racine du repo (`/Users/recarnot/dev/erom-agence-devil`), branche `feat/devil-review`.

## Exécution

- **Tasks 1 à 5** : exécutables par subagents (casting sonnet), une par une, review entre chaque.
- **Tasks 6 à 8** : SESSION PRINCIPALE uniquement (spawns de devils réels, décisions de Romain, dogfood interactif). Aucun subagent.

## Contrat de spawn agent (interface commune, référencée par les Tasks 3, 4, 6)

Prompt de spawn d'un devil (lignes exactes ; `FILES:`, `INTENT:`, `STACK:` seulement si les fichiers existent) :

```
MISSION_FILE=<abs>
SCHEMA_FILE=<abs>
VALIDATE_JQ=<expression jq -e sur une ligne>
INPUTS:
DIFF:<abs>
[FILES:<abs>]
[INTENT:<abs>]
[STACK:<abs>]

Exécute la procédure de transport.
```

Expression VALIDATE_JQ (UNE ligne, byte-identique à celle des skills code ; re-testée le 2026-08-18 : conforme = pass, issue sans failure_scenario = fail) :

```
has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.score|type=="number" and .>=0 and .<=100) and (.verdict|IN("approve","rework","reject")) and (.criteria|has("correctness") and has("architecture") and has("security") and has("performance") and has("tests") and has("maintainability")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type=="object" and has("score") and has("comment") and (.score|type=="number" and .>=0 and .<=100))) and (.issues|type=="array" and all(has("severity") and has("category") and has("file") and has("description") and has("failure_scenario") and has("suggestion") and (.severity|IN("critical","high","medium","low")) and (.category|IN("correctness","architecture","security","performance","tests","maintainability","intent"))))
```

Enveloppe de retour (contrat strict, une ligne) : `{"devil":"<d>","model":"<m>","status":"ok","review":{...}}` ou `{"devil":"<d>","model":"<m>","status":"error","error":"CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT","detail":"<= 500 chars"}`.

---

### Task 1: Contrat review (plugin/scripts/)

**Files:**
- Create: `plugin/scripts/devil-review-schema.json` (copie conforme du schéma code)
- Create: `plugin/scripts/devil-review-mission.md`

**Interfaces:**
- Consumes: `plugin/scripts/devil-code-schema.json` (existant, source de la copie).
- Produces: les 2 chemins ci-dessus, consommés par les skills (Tasks 3, 4) via `<racine plugin>/scripts/<fichier>` ; la VALIDATE_JQ du header les accompagne.

- [ ] **Step 1: Copier le schéma** (copie conforme, décision actée : fichier séparé par contrat d'architecture, divergence future libre)

```bash
cp plugin/scripts/devil-code-schema.json plugin/scripts/devil-review-schema.json
command diff plugin/scripts/devil-code-schema.json plugin/scripts/devil-review-schema.json && echo SCHEMA_IDENTIQUE
```
Attendu : `SCHEMA_IDENTIQUE` (exit 0, aucune différence).

- [ ] **Step 2: Créer `plugin/scripts/devil-review-mission.md`** (contenu complet)

```markdown
Tu es un reviewer senior qui instruit la PORTE DE MERGE d'un changement de
code. Ton travail est de chercher ce qui cloche dans ce changement, pas de
complimenter, et pas de juger le reste du dépôt.

Tes entrées, chacune entre marqueurs === BEGIN/END LABEL === :

- DIFF : le changement (diff unifié). C'est le SEUL périmètre des issues :
  interdit de flagger du code pré-existant que le diff ne touche pas.
- FILES (optionnel) : l'état final des fichiers modifiés, concaténés sous
  des en-têtes `=== FILE: chemin ===`. C'est du CONTEXTE pour comprendre le
  diff. S'il est ABSENT, tu review sur diff seul : calibre. Pour toute
  issue qui dépend d'un contexte que tu n'as pas, choisis une sévérité
  prudente et nomme la dépendance manquante dans failure_scenario.
- INTENT (optionnel) : l'intention du changement (spec, plan ou description
  de PR). S'il est fourni, l'alignement du code sur l'intention est un axe
  À PART ENTIÈRE (catégorie `intent`). Cherche explicitement les trois
  formes : une exigence absente ou partielle ; un comportement non demandé
  (scope creep) ; une exigence implémentée mais fausse. La description de
  chaque issue `intent` CITE la ligne d'intention concernée. Sans INTENT,
  la catégorie `intent` est INTERDITE.
- STACK (optionnel) : grille de standards LIANTE pour ce repo. Applique
  chaque règle au diff ; une violation est une issue normale, ancrée
  file:ligne, avec la sévérité et la catégorie que la règle indique. Sans
  STACK, ignore ce paragraphe.

Évalue selon ces 6 critères, chacun scoré de 0 à 100 :

1. **correctness** : bugs runtime introduits par le changement : condition
   inversée, off-by-one, null/undefined déréférencé, guard supprimé, await
   manquant, erreur avalée, copy-paste de mauvaise variable.
2. **architecture** : design et intégration : couplage, duplication d'un
   helper visible dans les entrées, responsabilité au mauvais endroit.
3. **security** : injections, authz manquante, secrets en dur, validation
   des entrées aux frontières réelles.
4. **performance** : complexité, requête dans une boucle (N+1), travail
   inutile dans un chemin chaud visible.
5. **tests** : le changement est-il testé ? Les tests vérifient-ils du
   comportement réel (assertions significatives) ?
6. **maintainability** : lisibilité réelle, conventions du fichier hôte,
   gestion d'erreurs.

Règles dures (anti-bruit) :
- Chaque issue DOIT porter : `file` au format `chemin:ligne` (ligne de
  l'état FINAL du fichier), une `description`, un `failure_scenario`, une
  `suggestion` actionnable.
- `failure_scenario` = la conséquence concrète et située :
  - correctness / security / performance / tests : des entrées concrètes
    et le résultat faux qu'elles produisent ;
  - architecture / maintainability : le coût futur nommé et son
    déclencheur (« ajouter un 4e transport forcera à dupliquer X dans
    3 skills »).
  Pas de conséquence concrète nommable = PAS d'issue.
- Style et naming purs INTERDITS (aucune issue « renommer », « reformater »).
- Ne flagge qu'à confiance élevée : uniquement ce que tu peux défendre.
- Jamais d'issue du type « vérifie que » ou « assure-toi que » : si tu ne
  peux pas nommer le défaut, il n'y a pas d'issue.
- Jamais d'explication du code à son auteur : il le connaît.
- Un même problème présent en plusieurs endroits = UNE issue, les autres
  localisations listées dans la description.
- Ne flagge pas ce que la porte déterministe du repo applique déjà (types,
  lint, format) : elle a tourné avant toi.
- Les fichiers de test modifiés se jugent au titre du critère `tests`, pas
  comme du code de production.
- Si le changement est excellent, dis-le : `issues` peut être vide.

Sévérité de chaque issue :
- `critical` : exploitable à distance, ou fuite / corruption de données, ou
  secret exposé : typiquement injection, contrôle d'accès cassé, credentials
  en clair. Une vraie faille exploitable se classe `critical`, pas `high`.
- `high` : bug de correction certain, ou risque sérieux sans exploitation
  directe démontrée. Un bug fonctionnel qui produit un comportement
  contraire à l'intention du changement se classe `high` minimum.
- `medium` / `low` : impact moindre, dette localisée.

Verdict : score >= 80 : "approve" ; 50 à 79 : "rework" ; < 50 : "reject".
"reject" signifie : le changement est structurellement mauvais, le réécrire
coûte moins cher que le corriger.
```

- [ ] **Step 3: Vérifier**

```bash
command jq . plugin/scripts/devil-review-schema.json >/dev/null && echo REVIEW_SCHEMA_OK
command grep -c 'STACK' plugin/scripts/devil-review-mission.md
command grep -c 'intent' plugin/scripts/devil-review-mission.md
command grep -c "$(printf '\xe2\x80\x94')" plugin/scripts/devil-review-mission.md
ls plugin/scripts/
```
Attendu : `REVIEW_SCHEMA_OK` ; grep STACK >= 3 ; grep intent >= 3 ; grep tiret cadratin = 0 (exit 1) ; `plugin/scripts/` contient les 8 fichiers contrats (`devil-{spec,brain,code,review}-{mission.md,schema.json}`) + `stacks/` à partir de la Task 2.

- [ ] **Step 4: Re-tester la VALIDATE_JQ contre le schéma livré** (2 échantillons)

```bash
TMP=$(mktemp -d "${TMPDIR:-/tmp}/devil-review-jq-XXXXXX")
cat > "$TMP/valid.json" <<'EOF'
{"score":72,"verdict":"rework","summary":"ok","criteria":{"correctness":{"score":70,"comment":"c"},"architecture":{"score":80,"comment":"c"},"security":{"score":60,"comment":"c"},"performance":{"score":75,"comment":"c"},"tests":{"score":65,"comment":"c"},"maintainability":{"score":80,"comment":"c"}},"issues":[{"severity":"high","category":"intent","file":"src/a.ts:42","description":"d","failure_scenario":"f","suggestion":"s"}]}
EOF
cat > "$TMP/invalid.json" <<'EOF'
{"score":72,"verdict":"rework","summary":"ok","criteria":{"correctness":{"score":70,"comment":"c"},"architecture":{"score":80,"comment":"c"},"security":{"score":60,"comment":"c"},"performance":{"score":75,"comment":"c"},"tests":{"score":65,"comment":"c"},"maintainability":{"score":80,"comment":"c"}},"issues":[{"severity":"high","category":"intent","file":"src/a.ts:42","description":"d","suggestion":"s"}]}
EOF
V='has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.score|type=="number" and .>=0 and .<=100) and (.verdict|IN("approve","rework","reject")) and (.criteria|has("correctness") and has("architecture") and has("security") and has("performance") and has("tests") and has("maintainability")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type=="object" and has("score") and has("comment") and (.score|type=="number" and .>=0 and .<=100))) and (.issues|type=="array" and all(has("severity") and has("category") and has("file") and has("description") and has("failure_scenario") and has("suggestion") and (.severity|IN("critical","high","medium","low")) and (.category|IN("correctness","architecture","security","performance","tests","maintainability","intent"))))'
command jq -e "$V" "$TMP/valid.json" >/dev/null && echo V_PASS || echo V_FAIL
command jq -e "$V" "$TMP/invalid.json" >/dev/null 2>&1 && echo I_PASS || echo I_FAIL
trash "$TMP"
```
Attendu : `V_PASS`, `I_FAIL`.

- [ ] **Step 5: Commit**

```bash
command git add plugin/scripts/devil-review-mission.md plugin/scripts/devil-review-schema.json
command git commit -m "feat(contracts): mission et schéma de l'exercice review"
```

---

### Task 2: Pack nextjs + fixtures stack (plugin/scripts/stacks/, examples/)

**Files:**
- Create: `plugin/scripts/stacks/nextjs.md`
- Create: `examples/review-stack.patch`
- Create: `examples/review-stack-files.txt`

**Interfaces:**
- Consumes: rien.
- Produces: le pack `nextjs` (input `STACK:` des Tasks 3, 4, 6) et un paquet fixture avec 3 violations de grille plantées : Server Action sans `actionClient` ni check de session (critical/security, bloc 1), mutation sans propriété dans le `where` (critical/security, bloc 1, IDOR), `z.string().email()` (high/correctness, bloc 2). Conventions du pack : starter web-stack-starter, CLAUDE.md et README vérifiés à l'exécution le 17/08/2026 (échantillon capturé, pas de mémoire).

- [ ] **Step 1: Créer `plugin/scripts/stacks/nextjs.md`** (contenu complet)

```markdown
Grille de standards LIANTE pour ce repo (stack : Next 16, Prisma 7,
Postgres, Zod 4, next-safe-action, Vitest, Playwright, Biome). Chaque
violation est une issue normale : ancrée file:ligne, failure_scenario
concret, sévérité et catégorie indiquées par la règle. Ne flagge que ce que
le DIFF introduit ou touche.

## Autorisation (violation : severity critical, category security)

- L'autorisation ne vit JAMAIS dans `middleware.ts` ni `proxy.ts`
  (CVE-2025-29927 : un header forgé contournait entièrement le middleware).
  Tout accès aux données passe par la DAL (`src/lib/dal.ts`).
- Trois checks distincts par mutation : session, permission, propriété.
  L'absence du check de PROPRIÉTÉ est LE défaut à chercher : c'est lui qui
  produit les IDOR.
- La propriété vit dans la clause `where` de la requête, jamais dans un
  `if` préalable (supprime aussi la course entre vérification et écriture).
- Toute Server Action est un endpoint POST public : aucune sans
  `actionClient` (next-safe-action, `src/lib/safe-action.ts`).
- Un fichier ne transite jamais par une Server Action : URL présignée,
  upload direct du client vers le stockage objet.

## Pièges de version (violation : severity high, category correctness)

- Prisma 7 : `url = env("DATABASE_URL")` dans le bloc datasource est un
  refus de validation (P1012). La connexion Migrate vit dans
  `prisma.config.ts`, `PrismaClient` reçoit un adapter (`@prisma/adapter-pg`).
  Aucune API du moteur Rust (supprimé en v7), aucune API Prisma 8 (RC).
- Zod 4 : `z.email()` et `z.url()` au top-level ; `z.string().email()` et
  `z.string().url()` n'existent plus.
- Next 16 : types de route globaux (`LayoutProps<"/">`, `PageProps<...>`).
  Pages Router, `getServerSideProps`, `getStaticProps` et `next/head` sont
  interdits.
- Node est requis au build ET à l'exécution ; Bun ne sert qu'à installer.
  `next start` ne fonctionne pas avec `output: "standalone"`.

## Architecture (violation : severity medium, category architecture)

- La logique métier vit dans la DAL et des fonctions pures, pas dans les
  composants.
- Un seul schéma Zod par formulaire, partagé client et serveur.
- `next/image`, jamais `<img>`.
- Aucune API Vercel-only (`@vercel/kv`, `@vercel/blob`, `@vercel/postgres`) :
  `output: "standalone"` et le Dockerfile doivent rester fonctionnels.
- Toute variable d'environnement passe par `src/env.ts` ; logs en JSON
  structuré, jamais de PII ni de secret dedans.

## Tests (violation : severity medium à high, category tests)

- Un Server Component async ne se rend pas sous Vitest (limite documentée
  Next) : un test Vitest qui le tente est une issue ; ces composants se
  testent sous Playwright.
- Antipatterns interdits : test change-detector (compteur ou snapshot de
  catalogue figé) et test qui asserte sur le contenu du code source. On
  assert sur le comportement.
- Le seed est déterministe : aucune valeur aléatoire non seedée dans une
  fixture.
```

- [ ] **Step 2: Créer `examples/review-stack.patch`** (contenu complet ; nouveau fichier unique, hunk `+1,18` : 18 lignes de contenu, arithmétique vérifiée)

```diff
diff --git a/src/app/actions.ts b/src/app/actions.ts
new file mode 100644
index 0000000..dd44ee5
--- /dev/null
+++ b/src/app/actions.ts
@@ -0,0 +1,18 @@
+"use server";
+
+import { z } from "zod";
+import { db } from "@/lib/db";
+
+const schema = z.object({
+  noteId: z.string(),
+  email: z.string().email(),
+});
+
+export async function updateNoteEmail(formData: FormData) {
+  const input = schema.parse(Object.fromEntries(formData));
+  await db.note.update({
+    where: { id: input.noteId },
+    data: { contactEmail: input.email },
+  });
+  return { ok: true };
+}
```

- [ ] **Step 3: Créer `examples/review-stack-files.txt`** (contenu complet, état final cohérent ligne à ligne avec le patch)

```text
=== FILE: src/app/actions.ts ===
"use server";

import { z } from "zod";
import { db } from "@/lib/db";

const schema = z.object({
  noteId: z.string(),
  email: z.string().email(),
});

export async function updateNoteEmail(formData: FormData) {
  const input = schema.parse(Object.fromEntries(formData));
  await db.note.update({
    where: { id: input.noteId },
    data: { contactEmail: input.email },
  });
  return { ok: true };
}
```

- [ ] **Step 4: Vérifier**

```bash
command grep -c '^+' examples/review-stack.patch
command grep -n 'z.string().email' examples/review-stack.patch
command grep -c "$(printf '\xe2\x80\x94')" plugin/scripts/stacks/nextjs.md
command grep -E -c 'AKIA[0-9A-Z]{16}|PRIVATE KEY' examples/review-stack.patch examples/review-stack-files.txt
```
Attendu : premier grep `19` (18 lignes de contenu + l'en-tête `+++`) ; deuxième : 1 ligne matchée ; troisième `0` (exit 1) ; quatrième `0` pour chaque fichier (exit 1, fixtures propres au scan).

- [ ] **Step 5: Commit**

```bash
command git add plugin/scripts/stacks/ examples/review-stack.patch examples/review-stack-files.txt
command git commit -m "feat(stacks): grille nextjs + fixture stack plantée (3 violations)"
```

---

### Task 3: Skill /erom-devil:review (unitaire)

**Files:**
- Create: `plugin/skills/review/SKILL.md`

**Interfaces:**
- Consumes: contrat Task 1, pack Task 2, « Contrat de spawn agent » + VALIDATE_JQ du header, étapes de `plugin/skills/code/SKILL.md` (référencées, jamais modifiées), agents existants `erom-devil:{gemini,glm,deepseek,opus,kimi}`.
- Produces: surface `/erom-devil:review [target] [intent.md] [devil] [stack=...] [no-gates]` ; Étapes 0 à 6, 9, 10, 11 consommées par référence par la Task 4.

- [ ] **Step 1: Créer `plugin/skills/review/SKILL.md`** (contenu complet ; la ligne VALIDATE_JQ du bloc de spawn est la version ÉCHAPPÉE, byte-identique à celle de `skills/code/SKILL.md` Étape 5 : la recopier depuis ce fichier, pas depuis ce plan)

````markdown
---
name: review
description: "Porte de merge d'un changement : porte déterministe (bun run check), chasse à l'intention, grille de stack optionnelle, review devil (Gemini par défaut, GLM, Deepseek, Opus, Kimi), vérification contradictoire par l'orchestrateur (Confirmée/Réfutée/Hypothèse), verdict GO/NO-GO, rapport persistant dans docs/reviews/. Triggers: /erom-devil:review, 'review de merge', 'porte de merge', 'review finale avant merge', 'prépare le merge'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Bash, Agent, AskUserQuestion, Edit, Write
---

# /erom-devil:review - La porte de merge

Gate de merge : session au tier pense recommandée (xhigh). `code` reste la
critique rapide et jetable en cours de route ; ici on instruit un dossier
complet : porte déterministe, intention, grille de stack, critique devil,
vérification contradictoire, verdict GO/NO-GO, rapport persistant. Plus
long et plus cher qu'un `/erom-devil:code` : on le paie une fois, avant le
merge vers main.

## Syntaxe

```
/erom-devil:review                     # auto : working tree sale, sinon branche vs base
/erom-devil:review main                # branche courante vs main
/erom-devil:review 123 glm             # PR GitHub, devil glm
/erom-devil:review main intent.md      # intention explicite
/erom-devil:review main stack=none     # débrayer la grille de stack
/erom-devil:review main no-gates       # sauter la porte déterministe (tracé)
```

Tokens reconnus et retirés dans cet ordre, le reste = target :
1. `no-gates` (littéral) : saute la porte déterministe, tracé au rapport.
2. `stack=<nom>` : force le pack (`stack=none` = aucun). Nom sans fichier
   `scripts/stacks/<nom>.md` correspondant : STOP avec la liste des packs.
3. dernier argument dans {gemini, glm, deepseek, opus, kimi} : devil
   (défaut `gemini`).
4. argument `*.md` existant : INTENT explicite (court-circuite la chasse).

## Étape 0 - Chemins du plugin

Racine du plugin : deux niveaux au-dessus du « Base directory for this
skill » injecté ci-dessus. Résous en absolu, vérifie l'existence (Read),
sinon STOP « plugin corrompu » :
- `MISSION_FILE` = `<racine>/scripts/devil-review-mission.md`
- `SCHEMA_FILE`  = `<racine>/scripts/devil-review-schema.json`
- `STACK_DIR`    = `<racine>/scripts/stacks/`

## Étape 1 - Résolution du target

Identique à `skills/code/SKILL.md` Étape 1 (mêmes formes, mêmes gardes et
cas limites, mêmes règles pour les fichiers non suivis), avec UNE exclusion
en plus : `docs/reviews/` sort du périmètre. Ajoute
`':(exclude)docs/reviews/'` au pathspec des commandes git de diff ; en mode
PR, retire du texte du diff les sections `diff --git` de ces chemins. Sans
cette exclusion, le rapport d'une review précédente entre dans le diff de
la suivante.

## Étape 2 - Porte déterministe

Applicable seulement si le diff reviewé correspond à l'arbre courant :
mêmes modes que la colonne « Correction guidée : oui » de la table FILES de
`code` Étape 2. Sinon : porte « non applicable (diff différent de l'arbre
courant) », mentionnée au rapport, continue.

1. `package.json` à la racine avec un script `check` : lance
   `bun run check` (timeout Bash 120000) et capture la sortie. Timeout
   dépassé = rouge, sortie partielle affichée.
2. Pas de script `check` : demande UNE fois la commande de porte à Romain
   (« aucune » accepté = porte sautée, tracée).
3. Verte : continue. Conserve les ~15 dernières lignes brutes de la sortie
   pour le rapport (recopiées, jamais résumées).
4. Rouge : STOP. « Porte rouge : corrige avant de convoquer une review ;
   les devils ne paient pas pour ce que tsc dit gratuitement. » Sortie
   brute affichée. `no-gates` outrepasse ; le rapport portera « porte :
   SAUTÉE sur décision » en tête de couverture.
5. Arbre sale en mode branche (porte applicable mais l'arbre n'est pas
   exactement le diff reviewé) : une ligne au rapport, rien de plus.

## Étape 3 - Chasse à l'intention

INTENT par priorité, premier trouvé gagne :
1. arg `*.md` explicite ;
2. body de la PR (mode PR), écrit dans un fichier temp ;
3. `.specs/<branche>/` ou `.specs/<slug proche>/` : un `*.md` de spec ;
4. `docs/superpowers/specs/*.md` dont le nom matche la branche ou le sujet ;
5. question à Romain (« aucune » accepté).

Plusieurs candidats aux niveaux 3-4 : liste-les, Romain choisit (ou
« aucune »). Sans INTENT : la catégorie `intent` est interdite au devil (la
mission le rappelle) et la couverture du rapport ouvre sur « review sans
intention fournie ». L'INTENT passe au scan anti-fuite comme tout input.

## Étape 4 - Packaging et sélection du pack

Packaging DIFF + FILES + INTENT : identique à `code` Étape 2 (mêmes règles,
mêmes budgets, même table FILES par mode), avec
`TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/review-XXXXXX")`.

Pack : si `stack=` est fourni, il décide. Sinon auto-détection :
`package.json` à la racine du repo reviewé avec `"next"` dans
`dependencies` : pack `nextjs`. Sinon aucun. Le pack sélectionné est
`$STACK_DIR/<nom>.md` : fichier du plugin, jamais copié, jamais scanné.

## Étape 5 - Scan anti-fuite

Identique à `code` Étape 3 (exclusions dures, regex, STOP sur hit, choix
exclure / annuler / forcer), sur DIFF + FILES + INTENT. Obligatoire, jamais
sauté.

## Étape 6 - Confirmation

> **Porte de merge :**
> - Mode : <PR 123 / branche vs main / range a..b / working tree>
> - Fichiers : <N> (±<lignes>) · FILES <complet / tronqué : n exclus / omis>
> - Porte : <verte (`bun run check`) / sautée (no-gates) / sautée (aucune
>   commande) / non applicable>
> - Intent : <chemin / body PR / aucune>
> - Stack : <nextjs / aucun>
> - Scan : <clean / forcé> · Correction guidée : <disponible / rapport seul>
> - Devil : <devil> (<modèle>)
>
> Je lance la review ?

Attends la confirmation de Romain (« oui », « go », « lance »).

## Étape 7 - Lancer le devil

Spawn `erom-devil:<devil>` (fallback `<devil>` sans préfixe si le type est
introuvable). INPUTS : `DIFF:` toujours ; `FILES:`, `INTENT:` et `STACK:`
seulement si les fichiers existent. Annonce avant le spawn :
« **Review en cours...** <devil> instruit le dossier (jusqu'à 9 min). »

```
Agent(
  subagent_type: "erom-devil:<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=<ligne échappée byte-identique à skills/code/SKILL.md Étape 5>\nINPUTS:\nDIFF:<abs diff>\nFILES:<abs files>\nINTENT:<abs intent>\nSTACK:<abs stack>\n\nExécute la procédure de transport."
)
```

Enveloppe `error` : gabarit d'échec de `code` Étape 6 (relance, autre
devil, review manuelle).

## Étape 8 - Ancrage

Identique à `code` Étape 6 : extraction des plages modifiées par fichier
depuis le DIFF, issue DÉCLASSÉE si fichier hors périmètre ou ligne hors
plages (tolérance ±3 ; fichier supprimé : ancrage au fichier seul). Les
DÉCLASSÉES vont en annexe « Non ancrées », jamais supprimées en silence,
exclues de la vérification, du verdict et de la correction guidée.

## Étape 9 - Passe de vérification (ton travail, pas celui du devil)

Lecture seule stricte : aucune mutation de l'arbre, commandes en lecture et
tests unitaires existants uniquement (jamais de migration, de seed ni
d'E2E).

Périmètre : toutes les issues `critical` et `high` ancrées. Budget
~2 minutes par issue : la vérification décisive la moins chère.

Pour chaque issue du périmètre :
1. relis le code réel au point ancré (Read, contexte large) et trace ce que
   le `failure_scenario` prétend ;
2. si une commande tranche (grep d'un symbole, test unitaire ciblé
   existant), lance-la et recopie la sortie ;
3. étiquette :
   - **Confirmée** : la trace ou la sortie prouve le scénario ; conserve la
     preuve courte (file:ligne + pourquoi, ou sortie recopiée) ;
   - **Réfutée** : tu as la preuve du contraire ; annexe du rapport avec la
     raison, exclue du verdict et de la correction guidée, jamais
     supprimée ;
   - **Hypothèse** : pas tranchable à coût raisonnable ; confiance
     haute / moyenne / basse + ce qui la trancherait.

Les issues hors périmètre (`medium`, `low`) gardent l'étiquette
**Non vérifiée** : elles restent au rapport, jamais promues ni supprimées.

**Balayage frontières** : une passe sur le DIFF ENTIER, quatre angles que
les reviews scopées au diff ratent :
1. entrée externe qui finit dans un chemin de fichier ou une requête sans
   validation ;
2. réponse HTTP d'erreur qui n'échoue pas en exception (`r.ok` jamais
   testé, status ignoré) ;
3. DX-breakage : variable d'environnement renommée ou supprimée, port
   remappé, étape de setup nouvelle obligatoire ;
4. copy user-visible qui ne matche plus le comportement.

Chaque finding frontière : étiquette « frontière (Claude) », Confirmée par
construction (tu l'as lue, cite file:ligne), mêmes règles de verdict.

## Étape 10 - Verdict

Sur les issues ancrées et vérifiées (devil + frontières ; Réfutées
exclues) :

| Situation | Verdict |
|---|---|
| >= 1 critical Confirmée | **NO-GO** (corriger avant merge) |
| >= 1 critical Hypothèse | **NO-GO**, levable par décision explicite tracée ; catégorie security : non levable tant que ni réfutée ni corrigée |
| >= 1 high Confirmée | **NO-GO**, levable par décision explicite tracée |
| high Hypothèse ou medium présentes (vérifiées ou non) | **GO AVEC RÉSERVES** (listées) |
| sinon (low seules ou rien) | **GO** |

Le verdict devil (score, approve/rework/reject) reste affiché : c'est la
matière du dossier, le verdict process est la décision. Une levée de NO-GO
s'écrit dans le rapport sous le verdict (date, raison de Romain) et le
frontmatter passe à `NO-GO LEVÉ`.

## Étape 11 - Rapport persistant

Écris (Write) `docs/reviews/<YYYY-MM-DD>-<slug>.md` dans le repo reviewé
(slug = branche ou target normalisé en kebab-case ; collision même jour,
re-review comprise : suffixe `-2`, `-3`). En français. Ne committe pas : le
commit rejoint le flux de merge. Structure exacte :

```markdown
---
date: <YYYY-MM-DD>
target: <mode et cible>
range: <base_sha>..<head_sha>
devils: <devil (modèle)>
porte: <bun run check : verte | sautée (no-gates) | aucune commande | non applicable>
intent: <source | aucune>
stack: <nextjs | aucun>
verdict: <GO | GO AVEC RÉSERVES | NO-GO | NO-GO LEVÉ>
---

# Review de merge - <cible>

## Verdict : <VERDICT>

<3 lignes : l'essentiel du dossier et la raison du verdict. Si NO-GO levé :
date et raison de Romain.>

## Couverture

<fichiers et lignes reviewés ; FILES complet / tronqué / omis ; ce qui n'a
PAS été couvert : E2E non relancés, exclusions du scan, budget FILES ;
« review sans intention fournie » le cas échéant ; arbre sale le cas
échéant.>

## Findings

| Statut | Sév | Cat | Fichier | Problème | Scénario | Preuve / à trancher |

<Confirmées (sévérité décroissante), puis Hypothèses (confiance
décroissante), puis Non vérifiées (sévérité décroissante). Findings
frontière étiquetés « frontière (Claude) ».>

## Annexes

### Réfutées
<issue + preuve de la réfutation ; « aucune » sinon>

### Non ancrées
<avec devil d'origine ; « aucune » sinon>

### Critères du devil
<les 6 critères scorés + commentaires>

### Porte déterministe
<commande + les dernières lignes brutes recopiées, ou la raison du saut>
```

Affiche ensuite dans le chat : le verdict, les comptes par statut
(Confirmées / Hypothèses / Réfutées / Non vérifiées / Non ancrées), le
chemin du rapport.

## Étape 12 - Correction guidée

Mêmes modes autorisés, même séquence et mêmes règles que `code` Étape 8
(Romain tranche issue par issue : appliquer / ignorer / stop ; `low`
ignorées sauf demande ; non-ancrées et Réfutées exclues ; le devil ne
modifie jamais rien, c'est toi qui édites). Ordre de passage : Confirmées
par sévérité décroissante, puis Hypothèses hautes. Après corrections :
re-review possible (max 2, même devil, mêmes target / intent / stack), la
porte est rejouée, chaque run produit son rapport suffixé.

## Règles

- Le devil ne modifie JAMAIS rien ; le rapport est écrit par toi.
- La passe de vérification ne mute JAMAIS l'arbre.
- Scan anti-fuite obligatoire, jamais sauté.
- `trash "$TMP_DIR"` en fin de run (succès comme échec).
- Un NO-GO levé se trace dans le rapport, jamais dans le seul chat.
- C'est une porte de merge, pas un lint : pas sur un diff de deux lignes
  (pour ça : `/erom-devil:code`).
````

- [ ] **Step 2: Recopier la VALIDATE_JQ échappée** : ouvrir `plugin/skills/code/SKILL.md`, copier la valeur `VALIDATE_JQ=...` du bloc Agent de son Étape 5 (version avec `\"` échappés), et remplacer le placeholder `<ligne échappée byte-identique à skills/code/SKILL.md Étape 5>` du bloc Agent de l'Étape 7 par cette valeur exacte.

- [ ] **Step 3: Vérifier**

```bash
command grep -n 'name: review' plugin/skills/review/SKILL.md | head -1
command grep -c '~/.claude' plugin/skills/review/SKILL.md
command grep -c "$(printf '\xe2\x80\x94')" plugin/skills/review/SKILL.md
command grep -c 'ligne échappée byte-identique' plugin/skills/review/SKILL.md
command grep -h 'VALIDATE_JQ=' plugin/skills/code/SKILL.md plugin/skills/review/SKILL.md | sort -u | wc -l
```
Attendu : ligne 2 matchée ; les greps 2, 3 et 4 rendent `0` (exit 1 : pas de chemin en dur, pas de tiret cadratin, placeholder remplacé) ; dernier compte = `1` (VALIDATE_JQ byte-identique à celle de code).

- [ ] **Step 4: Commit**

```bash
command git add plugin/skills/review/
command git commit -m "feat(skills): /erom-devil:review, la porte de merge unitaire"
```

---

### Task 4: Skill /erom-devil:review-swarm (tribunal)

**Files:**
- Create: `plugin/skills/review-swarm/SKILL.md`

**Interfaces:**
- Consumes: Task 3 (Étapes 0 à 6, 9, 10, 11 par référence), `plugin/skills/code-swarm/SKILL.md` Étapes 6-8 (quorum, consolidation, grille de verdict devils, par référence), agents `erom-devil:{gemini,glm,deepseek}`.
- Produces: surface `/erom-devil:review-swarm [target] [intent.md] [stack=...] [no-gates]`.

- [ ] **Step 1: Créer `plugin/skills/review-swarm/SKILL.md`** (contenu complet)

````markdown
---
name: review-swarm
description: "Porte de merge en tribunal : porte déterministe, chasse à l'intention, grille de stack, Gemini + GLM + Deepseek en parallèle, consolidation par problème de fond, vérification contradictoire par l'orchestrateur, verdict GO/NO-GO, rapport persistant dans docs/reviews/. Triggers: /erom-devil:review-swarm, 'tribunal de merge', 'review swarm avant merge'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Bash, Agent, AskUserQuestion, Edit, Write
---

# /erom-devil:review-swarm - La porte de merge en tribunal

Gate de merge : session au tier pense recommandée (xhigh). Les 3 devils
(gemini + glm + deepseek ; opus et kimi ne siègent pas, choix acté) jugent
le MÊME dossier en parallèle. Toi (l'orchestrateur) : tu consolides, tu
vérifies, tu tranches, tu écris le rapport.

## Étapes 0 à 6 - identiques à /erom-devil:review

Chemins plugin, résolution du target (+ exclusion `docs/reviews/`), porte
déterministe, chasse à l'intention, packaging + sélection du pack, scan
anti-fuite : déroule les Étapes 0 à 6 de `skills/review/SKILL.md` (un SEUL
TMP_DIR, partagé en lecture par les 3 spawns), à une différence près,
l'annonce de confirmation :

> **Tribunal de merge :** gemini + glm + deepseek en parallèle (jusqu'à
> 9 min). Porte : <état> · Intent : <source> · Stack : <pack> ·
> Cible : <mode/cible>. Je lance ?

Pas d'argument devil : le tribunal est fixe. `stack=`, `no-gates` et
`intent.md` s'appliquent comme en unitaire.

## Étape 7 - Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message. Prompt
IDENTIQUE pour les trois (celui de `/erom-devil:review` Étape 7 :
MISSION_FILE, SCHEMA_FILE, VALIDATE_JQ byte-identique, INPUTS
DIFF / FILES / INTENT / STACK selon existence), seul le subagent_type
change : `erom-devil:gemini`, `erom-devil:glm`, `erom-devil:deepseek`
(fallback sans préfixe si un type est introuvable).

## Étape 8 - Collecte, quorum, ancrage, consolidation

Par référence à `skills/code-swarm/SKILL.md` Étapes 6 et 7 :
- quorum : 3 voix pleines ; 2 voix : le rapport ouvre sur la voix absente ;
  1 voix ou moins : pas de verdict, rapport d'échec, proposer relance ou
  unitaire ;
- un retour qui n'est pas une enveloppe JSON valide compte comme voix
  absente (ne JAMAIS interpréter un texte d'erreur comme une review) ;
- ancrage PAR VOIX (les DÉCLASSÉES sortent avant consolidation, listées en
  « Non ancrées » avec leur devil) ;
- consolidation par PROBLÈME DE FOND (mêmes heuristiques : équivalence par
  fichier + plages voisines + même fond ; en cas de doute NE PAS
  fusionner ; badges 3/3, 2/3, 1/3 ; sévérité du groupe = la plus haute ;
  suggestion la plus actionnable conservée ; tri par convergence puis
  sévérité).

## Étape 9 - Passe de vérification

Identique à `/erom-devil:review` Étape 9 (vérification + balayage
frontières), appliquée UNE fois aux groupes consolidés. Périmètre élargi :
`critical` et `high`, plus les `medium` convergentes (2 voix et plus).

## Étape 10 - Verdict

D'abord la matière devils : grille VALABLE / MODIFICATIONS REQUISES /
JETABLE de `code-swarm` Étape 8, scores par devil + moyenne indicative,
voix dissonante jamais écrasée (reject isolé, écart > 30 sur un critère :
qui, sur quoi, son argument).

Garde-fou sécurité, version vérifiée : une issue `critical` / `security`
ANCRÉE vaut NO-GO si Confirmée ou Hypothèse ; si RÉFUTÉE avec preuve, le
garde-fou tombe et la réfutation figure en annexe du rapport.

Puis le verdict process : la table de `/erom-devil:review` Étape 10,
appliquée aux groupes vérifiés.

## Étape 11 - Rapport, correction, clôture

Rapport : même gabarit que `/erom-devil:review` Étape 11, enrichi swarm :
- frontmatter `devils: gemini + glm + deepseek` (voix absente notée) ;
- colonne `Conv` (badge de convergence) dans le tableau Findings ;
- annexe « Voix dissonantes » ;
- annexe « Scores devils » : grille par devil + moyenne indicative.

Affichage chat : verdict, comptes par statut, chemin du rapport.

Correction guidée : mêmes règles que `/erom-devil:review` Étape 12, ordre
de passage : groupes convergents (3/3, 2/3) Confirmés d'abord, puis
critical / high isolées Confirmées, puis Hypothèses hautes. Re-swarm max 1,
porte rejouée, rapport suffixé. `trash "$TMP_DIR"` en fin de run.

## Règles

- Toutes celles de `/erom-devil:review`, plus :
- opus et kimi ne siègent pas au tribunal ; second avis indépendant en
  unitaire : `/erom-devil:review <target> opus`.
- Coût : 3 modèles en parallèle + une passe de vérification. C'est LA porte
  avant merge to main ; pour une critique en cours de route :
  `/erom-devil:code(-swarm)`.
````

- [ ] **Step 2: Vérifier**

```bash
command grep -n 'name: review-swarm' plugin/skills/review-swarm/SKILL.md | head -1
command grep -c '~/.claude' plugin/skills/review-swarm/SKILL.md
command grep -c "$(printf '\xe2\x80\x94')" plugin/skills/review-swarm/SKILL.md
command grep -c 'opus' plugin/skills/review-swarm/SKILL.md
```
Attendu : ligne 2 matchée ; greps 2 et 3 rendent `0` (exit 1) ; grep opus >= 2 (l'exclusion du tribunal est écrite).

- [ ] **Step 3: Commit**

```bash
command git add plugin/skills/review-swarm/
command git commit -m "feat(skills): /erom-devil:review-swarm, le tribunal de merge"
```

---

### Task 5: Manifest 0.7.0, README, _memory_

**Files:**
- Modify: `plugin/.claude-plugin/plugin.json`
- Modify: `plugin/README.md`
- Modify: `_memory_/architecture.md`, `_memory_/key-files.md`, `_memory_/patterns.md`

**Interfaces:**
- Consumes: tout ce qui précède.
- Produces: plugin v0.7.0 auto-cohérent, mémoire projet à jour, prêt pour les smokes.

- [ ] **Step 1: `plugin/.claude-plugin/plugin.json`, contenu complet** (version 0.7.0, description et keywords étendus ; le reste inchangé)

```json
{
  "$schema": "https://www.schemastore.org/claude-code-plugin-manifest.json",
  "name": "erom-devil",
  "description": "Avocats du diable : reviews critiques de specs (score, verdict, issues), interrogatoire socratique de brainstormings (les questions jamais posées), review de changements de code (PR, branche, range, working tree) et porte de merge complète (porte déterministe, intention, grille de stack, vérification contradictoire, verdict GO/NO-GO, rapport persistant), par Gemini, GLM, Deepseek, Opus et Kimi, unitaires ou en swarm.",
  "version": "0.7.0",
  "author": {
    "name": "Romain Ecarnot",
    "url": "https://github.com/eRom"
  },
  "repository": "https://github.com/eRom/erom-devil",
  "license": "MIT",
  "keywords": ["review", "specs", "brainstorming", "code-review", "merge-gate", "devils-advocate", "socratic", "gemini", "glm", "deepseek", "opus", "kimi", "swarm"],
  "skills": "./skills/"
}
```

- [ ] **Step 2: `plugin/README.md`, quatre modifications** (anchors : le README actuel liste trois angles dans l'intro, un tableau des devils, une section Usage, des paragraphes spec / brain / code ; les blocs existants contiennent des tirets cadratins : ne PAS les recopier, seulement insérer les blocs neufs ci-dessous)

Modification 1 : dans la liste des angles de l'intro (après la puce **code**), ajouter :

```markdown
- **review** : la porte de merge complète : porte déterministe, chasse à
  l'intention, grille de stack optionnelle, review devil, vérification
  contradictoire par l'orchestrateur, verdict GO/NO-GO, rapport persistant
  dans `docs/reviews/`.
```

Modification 2 : dans le bloc Usage, après les lignes `/erom-devil:code-swarm`, ajouter :

```
/erom-devil:review                        # porte de merge, devil gemini
/erom-devil:review main glm               # branche vs main, devil glm
/erom-devil:review main stack=none        # sans grille de stack
/erom-devil:review-swarm main             # tribunal de merge + vérification
```

Modification 3 : après le paragraphe descriptif **code**, ajouter :

```markdown
**review** : entrée : un changement (mêmes cibles que code), plus une porte
déterministe (`bun run check` si le script existe, STOP si rouge), une
chasse à l'intention (arg, body PR, `.specs/`, `docs/superpowers/specs/`),
et une grille de stack optionnelle (`scripts/stacks/nextjs.md`,
auto-détectée). Sortie par devil : même JSON que code. Ensuite
l'orchestrateur vérifie chaque critical/high contre le code réel
(Confirmée / Réfutée / Hypothèse, preuve exigée), balaye quatre frontières
que les reviews scopées au diff ratent, tranche GO / GO AVEC RÉSERVES /
NO-GO, et écrit un rapport persistant dans `docs/reviews/` du repo reviewé.
Répartition des rôles : `code` est la critique rapide et jetable à chaque
phase ; `review` est la porte avant merge to main.
```

Modification 4 : dans la table des devils, aucune modification (transports
inchangés). Vérifier seulement que rien ne contredit la nouvelle surface.

- [ ] **Step 3: `_memory_/architecture.md`** : mettre à jour la ligne d'en-tête `> MàJ :` avec la date du jour et v0.7.0 ; dans la liste des exercices, ajouter après le bloc **code** :

```markdown
- **review** : porte de merge complète (4e exercice, v0.7.0) : porte
  déterministe avant tout appel modèle, chasse à l'intention, input STACK
  (grille `scripts/stacks/nextjs.md`), passe de vérification orchestrateur
  (Confirmée/Réfutée/Hypothèse + balayage frontières), verdict GO/NO-GO,
  rapport persistant `docs/reviews/`. Skills review{,-swarm} par référence
  aux étapes de code{,-swarm} ; mission/schéma propres ; agents inchangés.
```

et dans le bloc arborescence, ajouter les lignes :

```
plugin/scripts/devil-review-{mission.md,schema.json}  exercice review
plugin/scripts/stacks/nextjs.md                       grille de stack Next 16 / Prisma 7
plugin/skills/review{,-swarm}/                        porte de merge (unitaire, tribunal)
```

- [ ] **Step 4: `_memory_/key-files.md` et `_memory_/patterns.md`** : lire chaque fichier, puis ajouter dans la section adéquate (fin de table ou de liste existante) :

key-files.md :
```markdown
- `plugin/scripts/devil-review-mission.md` : mission porte de merge (STACK, axe intent, anti-bruit renforcé)
- `plugin/scripts/stacks/nextjs.md` : grille de stack liante (autorisation DAL, pièges Prisma 7 / Zod 4 / Next 16, tests)
- `plugin/skills/review/SKILL.md` : porte de merge unitaire (étapes 0 à 12, verdict GO/NO-GO, rapport docs/reviews/)
- `plugin/skills/review-swarm/SKILL.md` : tribunal de merge (3 voix + vérification orchestrateur)
```

patterns.md :
```markdown
- **Passe de vérification post-devils (review)** : chaque critical/high
  ancrée est vérifiée contre le code réel avant verdict (Confirmée avec
  preuve / Réfutée en annexe / Hypothèse avec confiance) ; leçon spec-swarm
  institutionnalisée. Le garde-fou sécurité tombe UNIQUEMENT sur réfutation
  prouvée.
- **Porte déterministe avant modèle (review)** : `bun run check` d'abord,
  STOP si rouge ; les devils ne paient jamais ce que tsc dit gratuitement.
```

- [ ] **Step 5: Vérifier**

```bash
command jq -r '.version' plugin/.claude-plugin/plugin.json
command grep -c 'review' plugin/README.md
EM=$(printf '\xe2\x80\x94')
command git show HEAD:plugin/README.md | command grep -c "$EM"
command grep -c "$EM" plugin/README.md
```
Attendu : `0.7.0` ; grep review >= 8 ; les deux derniers comptes sont ÉGAUX (les blocs neufs n'introduisent aucun tiret cadratin ; l'existant n'est pas réécrit).

- [ ] **Step 6: Commit**

```bash
command git add plugin/.claude-plugin/plugin.json plugin/README.md _memory_/
command git commit -m "chore(0.7.0): manifest, README et mémoire pour l'exercice review"
```

---

### Task 6: Smoke transport (SESSION PRINCIPALE, aucun commit)

**Files:** aucun (vérification pure).

**Interfaces:**
- Consumes: Tasks 1, 2 ; agent `erom-devil:glm` (le moins cher) ; fixtures `examples/`.

- [ ] **Step 1: Smoke mission sans STACK** : spawn direct `erom-devil:glm` (timeout Bash côté agent 540000, annonce longue durée) avec le contrat du header et ces INPUTS (chemins absolus du repo) :
  - `DIFF:<repo>/examples/code-diff.patch`
  - `FILES:<repo>/examples/code-files.txt`
  - MISSION_FILE = `<repo>/plugin/scripts/devil-review-mission.md`, SCHEMA_FILE = `<repo>/plugin/scripts/devil-review-schema.json`, VALIDATE_JQ du header.

Attendu : enveloppe `status:"ok"` ; parmi les issues, l'injection SQL de `src/db.ts` en `critical`/`security` ; aucune issue de style ; `failure_scenario` partout ; aucune issue de catégorie `intent` (INTENT absent) ; ancrage 100 % dans les plages du patch (vérifier à la main sur 2 issues).

- [ ] **Step 2: Smoke mission avec STACK** : même spawn, INPUTS :
  - `DIFF:<repo>/examples/review-stack.patch`
  - `FILES:<repo>/examples/review-stack-files.txt`
  - `STACK:<repo>/plugin/scripts/stacks/nextjs.md`

Attendu : enveloppe `ok` ; au moins une issue `critical`/`security` ancrée sur `src/app/actions.ts` (Server Action sans actionClient, ou mutation sans propriété dans le where) ; `z.string().email()` attrapée (high/correctness attendue, tolérer medium) ; zéro issue sur du code hors diff.

- [ ] **Step 3: En cas d'échec d'enveloppe** (CLI_FAILED, TIMEOUT) : relancer UNE fois ; deux échecs = STOP et diagnostic transport (gotchas du repo), pas de contournement.

---

### Task 7: Probes de la porte déterministe (SESSION PRINCIPALE, aucun commit)

**Files:** aucun (repo jetable).

- [ ] **Step 1: Construire le repo jetable**

```bash
R=$(mktemp -d "${TMPDIR:-/tmp}/review-probe-XXXXXX")
cd "$R" && command git init -q
printf '{"name":"probe","scripts":{"check":"bun -e \"process.exit(1)\""}}' > package.json
echo 'const x = 1;' > a.ts
command git add -A && command git commit -qm init
command git switch -qc feat/probe
echo 'const y = 2;' >> a.ts
command git add -A && command git commit -qm change
```

- [ ] **Step 2: Probe porte rouge** : depuis `$R`, dérouler `/erom-devil:review main` (manuellement : suivre la skill). Attendu : STOP à l'Étape 2 avec la sortie de `bun run check` affichée, AUCUN appel modèle, AUCUN fichier `docs/reviews/` créé.

- [ ] **Step 3: Probe no-gates** : dérouler `/erom-devil:review main no-gates` jusqu'à l'Étape 6 (confirmation) puis répondre « annule ». Attendu : la confirmation affiche « Porte : sautée (no-gates) », « Stack : aucun » (pas de next dans package.json), « Intent : aucune » après chasse infructueuse ; l'annulation nettoie (`trash "$TMP_DIR"`), aucun appel modèle.

- [ ] **Step 4: Nettoyage**

```bash
cd /Users/recarnot/dev/erom-agence-devil && trash "$R"
```

---

### Task 8: Dogfood méta, clôture du chantier (SESSION PRINCIPALE, gates Romain)

- [ ] **Step 1: Dogfood** : depuis le repo devil sur `feat/devil-review`, lancer `/erom-devil:review-swarm main`. La chasse à l'intention doit trouver `.specs/plugin-devil-review/` (candidats listés, choisir `architecture-technique.md`) ; la porte est « sautée (aucune commande) » (pas de package.json) ; pack aucun. Dérouler jusqu'au rapport : premier fichier de `docs/reviews/` du repo devil. Vérifier : frontmatter complet, tri des Findings, annexes présentes, comptes affichés en chat. Si un re-swarm a lieu : vérifier que le rapport précédent n'apparaît PAS dans le nouveau DIFF (preuve du pathspec `':(exclude)docs/reviews/'`).
- [ ] **Step 2: Traiter le verdict** : corrections via la correction guidée si NO-GO ou réserves sérieuses (décisions de Romain), re-swarm si corrections (max 1), committer les fixes et le rapport.
- [ ] **Step 3: Vérifier l'attendu de réfutation** : au moins une issue devil étiquetée Réfutée avec preuve pendant le dogfood ; si aucune (tribunal parfait), le noter dans le rapport de chantier, ne pas fabriquer. Limite actée (esquive visible) : le chemin « garde-fou sécurité tombé sur réfutation prouvée » n'est pas testable en fixture à coût raisonnable ; il se valide par relecture de la logique du SKILL.md et à sa première occurrence réelle.
- [ ] **Step 4: GATE Romain : merge** `feat/devil-review` dans `main` (après son feu vert explicite).
- [ ] **Step 5: GATE Romain : livraison** via `/plugin-release` (bump marketplace, push, CI, uninstall + install, smoke via plugin installé).
- [ ] **Step 6: GATE Romain : retouches starter** (repo `/Users/recarnot/dev/web-stack-starter`) : `trash .claude/skills/code-review/` (copie Matt non adaptée) ; dans `CLAUDE.md` section Vérification, ajouter : « Avant merge vers main : `/erom-devil:review-swarm main` (tribunal + vérification + rapport dans `docs/reviews/`). » Commit starter.
- [ ] **Step 7: Proposer à Romain** (jamais d'office) la retouche de son CLAUDE.md global : la ligne passe-sécu de la section Qualité mentionne `/erom-devil:review-swarm` comme couverture du gate de merge, `/security-review` restant l'outil spécialisé sur demande.
- [ ] **Step 8: Mémoire de clôture** : promouvoir la note de chantier (statut implemented), MàJ `_memory_/` si le dogfood a appris quelque chose, retro courte si Romain la demande.
