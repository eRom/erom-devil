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
