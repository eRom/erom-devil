# Architecture technique — devil code (plugin devil v0.3.0)

> 2026-07-18. Complète `brainstorming.md` (les décisions produit y font foi).

## 1. Arborescence cible

```
agents/devil-{gemini,glm,deepseek,opus}.md   INCHANGÉS (transport pur v0.2.x)
scripts/devil-code-mission.md                NOUVEAU : mission review de code
scripts/devil-code-schema.json               NOUVEAU : schéma verdict code
skills/devil-code/SKILL.md                   NOUVEAU : unitaire (arg devil, gemini défaut, opus inclus)
skills/devil-code-swarm/SKILL.md             NOUVEAU : tribunal gemini+glm+deepseek (opus exclu)
examples/code-diff.patch                     NOUVEAU : fixture diff planté (défauts connus)
examples/code-files.txt                      NOUVEAU : fixture paquet FILES pré-assemblé
examples/code-secret.patch                   NOUVEAU : fixture scan pré-vol (faux secret)
```

Versioning : `plugin.json` → 0.3.0 ; marketplace bump ; uninstall + install.
Aucune modification des 4 agents ni des exercices spec/brain.

## 2. Contrat transport (rappel, inchangé)

Le spawn suit le contrat v0.2.0 : `MISSION_FILE` + `SCHEMA_FILE` +
`VALIDATE_JQ` + `INPUTS` (lignes `LABEL:CHEMIN_ABSOLU`, 1 à N). Contenus
embarqués pour glm/deepseek/opus (run hermétique), chemins lus par agy pour
gemini. Enveloppe `{devil, model, status, review|error}`, erreurs
`CLI_FAILED|PARSE_ERROR|SCHEMA_INVALID|TIMEOUT`, timeout Bash 540000 ms,
1 retry, TMP_DIR par run, `trash`.

Appels par exercice (table complétée) :

| Skill | MISSION_FILE | SCHEMA_FILE | INPUTS |
|---|---|---|---|
| devil-spec(-swarm) | devil-spec-mission.md | devil-spec-schema.json | `BRAINSTORMING:` + `SPECS:` |
| devil-brain(-swarm) | devil-brain-mission.md | devil-brain-schema.json | `BRAINSTORMING:` |
| devil-code(-swarm) | devil-code-mission.md | devil-code-schema.json | `DIFF:` (+ `FILES:` + `INTENT:` optionnels) |

Taille : un paquet FILES ≤ 200 Ko ≈ 50 k tokens, dans le contexte des
modèles cloud utilisés. Un dépassement de contexte se solde par une erreur
franche du transport (enveloppe `error`), posture v0.2.0 assumée.

## 3. Résolution du target (orchestrateur, étape 1)

Arguments : `[target] [intent.md] [devil]`. Dernier arg ∈
`gemini|glm|deepseek|opus` → devil (défaut `gemini`) ; un arg `*.md`
existant → INTENT ; le reste → target :

| Forme | Mode | Résolution du diff |
|---|---|---|
| nombre pur (`123`) | PR | `gh pr view 123 --json title,body,baseRefName,headRefName,state,additions,deletions,changedFiles` + `gh pr diff 123` |
| contient `..` (`a..b`) | range | `git diff a..b` |
| ref valide (`main`, `HEAD~1`, sha — `git rev-parse --verify`) | branche vs base | `git diff ref...HEAD` |
| *(rien)*, tree sale (`git status --porcelain` non vide) | working tree | `git diff HEAD` (staged + unstaged) |
| *(rien)*, tree propre | branche vs base auto | base = HEAD branch d'`origin` (fallback `main` puis `master` locale) → `git diff base...HEAD` |

Gardes : hors repo git → stop ; ref/PR introuvable → stop avec le message
git/gh ; diff vide → stop « rien à reviewer », aucun appel modèle.

INTENT par priorité : arg `*.md` > body de la PR (mode PR, écrit en temp) >
absent. Pas d'auto-detect `.specs/`.

## 4. Packaging (étape 2)

`TMP_DIR=$(mktemp -d …/devil-code-XXXXXX)`, trois fichiers :

- **DIFF** : le diff unifié brut, jamais tronqué. > 1 Mo → refus et
  suggestion de découper (range plus petit). Fichiers binaires : hunks
  absents par nature (`Binary files differ`), listés comme tels.
- **FILES** : état final de chaque fichier texte modifié, chacun sous
  `=== FILE: chemin ===` ; fichiers supprimés listés `=== FILE: chemin
  (supprimé) ===` sans contenu. Source selon le mode :

| Mode | Source FILES | Correction guidée |
|---|---|---|
| working tree | `cat` du checkout | oui |
| branche vs base (tree propre) | `cat` du checkout (HEAD) | oui |
| range `a..b` avec `b` = HEAD | `cat` du checkout | oui |
| range historique (`b` ≠ HEAD) | `git show b:chemin` | non — rapport seul |
| PR checkoutée localement | `cat` du checkout | oui |
| PR non checkoutée | **FILES omis** + badge « REVIEW SUR DIFF SEUL — contexte réduit » | non — rapport seul |

  Budget FILES : 200 Ko. Au-delà : tri des fichiers par lignes de diff
  décroissantes, on garde jusqu'au budget, les exclus sont LISTÉS en tête du
  paquet (`=== FILES EXCLUS (budget) : … ===`) et signalés dans la
  confirmation.
- **INTENT** *(optionnel)* : copie du doc d'intention ou body PR.

## 5. Scan pré-vol anti-fuite (étape 3, dogfood Q2 — 3/3)

Ordre : exclusions de fichiers d'abord, scan de contenu ensuite, sur DIFF +
FILES + INTENT assemblés.

**Exclusions dures du paquet FILES** (glob, liste figée dans la skill) :
`.env*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `id_rsa*`, `id_ed25519*`,
`credentials*`, `secrets*`, `*.keystore`. Fichier exclu = listé dans la
confirmation ; ses hunks restent dans le DIFF → le scan de contenu ci-dessous
les couvre.

**Regex haute valeur** (`grep -E -n -i` sauf mention, liste figée) :

| Cible | Pattern |
|---|---|
| Clés privées | `-----BEGIN [A-Z ]*PRIVATE KEY` |
| AWS access key | `AKIA[0-9A-Z]{16}` (sensible casse) |
| GitHub tokens | `ghp_[A-Za-z0-9]{36}` · `github_pat_[A-Za-z0-9_]{22,}` |
| Clés style sk- | `sk-[A-Za-z0-9_-]{20,}` |
| Slack | `xox[baprs]-[A-Za-z0-9-]{10,}` |
| Google | `AIza[0-9A-Za-z_-]{35}` |
| JWT | `eyJ[A-Za-z0-9_-]{20,}\.eyJ` |
| Affectations | `(password|passwd|secret|token|api[_-]?key|auth)\s*[:=]\s*['"][^'"]{8,}` |
| URL à credentials | `[a-z][a-z0-9+.-]*://[^/\s:]+:[^@\s]+@` |

Un hit → **STOP avant tout envoi** : lignes suspectes affichées
(fichier:ligne + extrait), Romain tranche — exclure le fichier, annuler,
ou forcer en connaissance de cause. Le choix « forcer » est rappelé dans la
confirmation. Biais assumé : le faux positif (un STOP à tort coûte une
relecture, un faux négatif coûte une fuite irréversible).

## 6. Contrat code

### Mission (`devil-code-mission.md`)

Rôle : reviewer senior qui joue l'avocat du diable sur un **changement** de
code. Structure imposée par le transport : les inputs arrivent entre
marqueurs `=== BEGIN/END LABEL ===`.

- **DIFF** — le changement. Seul périmètre des issues : interdit de flagger
  du code pré-existant non touché par le diff.
- **FILES** — contexte : état final des fichiers modifiés, concaténés sous
  en-têtes `=== FILE: chemin ===`. Peut être ABSENT (mode dégradé) : alors
  calibrer — issue dépendant du contexte absent → sévérité prudente et
  dépendance nommée dans `failure_scenario`.
- **INTENT** — optionnel : l'intention du changement. S'il est fourni, juger
  aussi l'alignement code ↔ intention (catégorie `intent`) : dérives, oublis,
  ajouts non demandés. Sans INTENT, la catégorie `intent` est interdite.

Six critères scorés 0-100 : `correctness` (bugs runtime : conditions
inversées, off-by-one, null/undefined, guards supprimés, await manquant,
erreurs avalées…), `architecture` (design, intégration, couplage),
`security` (injections, authz, secrets, validation des entrées),
`performance` (complexité, N+1, allocations dans les boucles),
`tests` (le changement est-il testé ? les tests vérifient-ils du
comportement réel ?), `maintainability` (lisibilité, conventions du fichier
hôte, error handling).

Anti-bruit (règles dures) :
- Chaque issue : `file` au format `chemin:ligne` (ligne du fichier à l'état
  final) + `failure_scenario` + `suggestion` actionnable.
- `failure_scenario` = « conséquence concrète et située », par catégorie :
  correctness/security/performance/tests → entrées concrètes → résultat
  faux ; architecture/maintainability → coût futur nommé + déclencheur
  (« ajouter un 4e transport forcera à dupliquer X dans 3 skills »).
  Pas de conséquence concrète nommable = pas d'issue.
- Style et naming purs interdits. Confiance élevée exigée : ne flagger que
  ce qu'on peut défendre.
- Fichiers de test modifiés : reviewés au titre du critère `tests`, pas
  comme du code de production.

Verdict : score ≥ 80 → `approve` ; 50-79 → `rework` ; < 50 → `reject`
(le changement est structurellement mauvais : réécrire coûte moins cher que
corriger). Sortie : UN objet JSON conforme au schéma, rien d'autre.

### Schéma (`devil-code-schema.json`)

```json
{
  "score": "int 0-100",
  "verdict": "approve | rework | reject",
  "summary": "2-3 phrases",
  "criteria": {
    "correctness":     { "score": "int 0-100", "comment": "string" },
    "architecture":    { "score": "int 0-100", "comment": "string" },
    "security":        { "score": "int 0-100", "comment": "string" },
    "performance":     { "score": "int 0-100", "comment": "string" },
    "tests":           { "score": "int 0-100", "comment": "string" },
    "maintainability": { "score": "int 0-100", "comment": "string" }
  },
  "issues": [
    {
      "severity": "critical | high | medium | low",
      "category": "correctness | architecture | security | performance | tests | maintainability | intent",
      "file": "chemin/fichier.ext:ligne",
      "description": "le problème",
      "failure_scenario": "conséquence concrète et située (sémantique par catégorie)",
      "suggestion": "correction actionnable"
    }
  ]
}
```

Tous les champs requis à chaque niveau ; `issues` peut être vide. Enums en
anglais, contenus en français (convention plugin).

### VALIDATE_JQ (renforcé, dogfood Q3)

Une ligne, posée en single quotes par l'agent :

```
has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.score|type=="number" and .>=0 and .<=100) and (.verdict|IN("approve","rework","reject")) and (.criteria|has("correctness") and has("architecture") and has("security") and has("performance") and has("tests") and has("maintainability")) and (.issues|type=="array" and all(has("severity") and has("category") and has("file") and has("description") and has("failure_scenario") and has("suggestion") and (.severity|IN("critical","high","medium","low")) and (.category|IN("correctness","architecture","security","performance","tests","maintainability","intent"))))
```

### Ancrage vérifié (orchestrateur, dogfood Q3)

Après réception d'une enveloppe `ok`, l'orchestrateur extrait la liste des
fichiers du DIFF (`git diff --name-only` du même target) et vérifie le champ
`file` de chaque issue : fichier hors périmètre → issue **DÉCLASSÉE**,
affichée dans une section « Non ancrées » sous le tableau (jamais supprimée
en silence), exclue de la correction guidée et du garde-fou sécurité. En
swarm, l'ancrage est appliqué PAR VOIX, avant consolidation.

## 7. Skills code

### `/devil-code [target] [intent.md] [devil]` (unitaire)

1. Étape 0 v0.2.0 : racine plugin = deux niveaux au-dessus du base dir ;
   `MISSION_FILE`/`SCHEMA_FILE` résolus et vérifiés.
2. Résolution target/intent/devil (§ 3), packaging (§ 4), scan pré-vol
   (§ 5).
3. Confirmation systématique :
   > **Contexte détecté :** mode <PR/branche/range/working tree> · <cible> ·
   > <N> fichiers (±<lignes>) · FILES <complet/tronqué : n exclus/omis> ·
   > intent <chemin/body PR/absent> · scan <clean/forcé> · devil <devil>
   > Je lance la review ?
4. Spawn `devil:devil-<devil>` (fallback sans préfixe), annonce « jusqu'à
   9 min ».
5. Enveloppe `error` → gabarit d'échec v0.2.0 (relance, autre devil, review
   manuelle). Enveloppe `ok` → ancrage (§ 6) puis rapport :

```
══════ CODE REVIEW ══════
[REVIEW SUR DIFF SEUL — contexte réduit]        ← si FILES omis

Devil : <devil> (<model>) · Cible : <mode/cible>
Score : <score>/100  [APPROVE ✓ | REWORK ✗ | REJECT ☠]

Critères :   Correctness / Architecture / Security / Performance / Tests / Maintainability
Résumé : <summary>

| Sév | Cat | Fichier | Problème | Scénario | Suggestion |
[section « Non ancrées » le cas échéant]
```

6. Next steps par verdict, gabarits devil-spec (approve → prêt à
   commit/merge ; rework → corriger/ignorer/re-review ; reject → pas de
   correction incrémentale, on rouvre la conception).
7. Correction guidée UNIQUEMENT si le mode l'autorise (table § 4) : sévérité
   décroissante, `low` ignorées sauf demande, relecture du fichier avant
   chaque Edit, max 2 re-review, le devil ne modifie jamais rien.
8. `trash "$TMP_DIR"`.

### `/devil-code-swarm [target] [intent.md]` (tribunal)

1. Résolution + packaging + scan UNE fois (TMP_DIR orchestrateur partagé en
   lecture par les 3 agents ; chaque agent garde son TMP_DIR interne).
2. Spawn des 3 devils (`gemini`, `glm`, `deepseek` — opus exclu, décision
   actée) en UN SEUL message, prompt identique.
3. Quorum v0.2.0 : 3 voix pleines ; 2 voix → rapport ouvert sur la voix
   absente ; ≤ 1 voix → pas de verdict, rapport d'échec.
4. Ancrage par voix, puis consolidation par PROBLÈME DE FOND (jugement
   sémantique aidé par file:ligne — même fichier et lignes voisines =
   candidats à l'équivalence, mais deux issues au même endroit peuvent être
   deux problèmes distincts). Badges 3/3, 2/3, 1/3 ; sévérité du groupe = la
   plus haute ; suggestion la plus actionnable.
5. **Verdict** (dogfood Q1 — 3/3) :

| Situation (voix exprimées) | Verdict |
|---|---|
| ≥ 2 approve ET 0 reject | **VALABLE** |
| ≥ 2 reject | **JETABLE** |
| tout le reste (dont split approve/rework/reject) | **MODIFICATIONS REQUISES** |

   **Garde-fou sécurité** : ≥ 1 issue `critical` de catégorie `security`
   ANCRÉE portée par au moins un devil → verdict plafonné à MODIFICATIONS
   REQUISES, même à 3 approve, avec mention explicite du garde-fou dans le
   rapport. Pas de score consolidé : grille des scores par devil + moyenne
   indicative.
6. Voix dissonante (reject isolé, écart > 30 sur un critère) : section
   dédiée, jamais écrasée. Rapport façon spec-swarm + colonne Fichier +
   badge DIFF seul le cas échéant.
7. Next steps par verdict ; correction selon la table § 4 ; max 1 re-swarm.

## 8. Vérification (pas de suite de tests classique)

1. **Fixtures plantées** (`examples/code-diff.patch` + `code-files.txt`) :
   un diff avec 5 défauts connus — null deref (correctness), injection
   (security, critical), N+1 (performance), duplication de helper
   (architecture), test assertion-free (tests). Smoke agent direct (glm, le
   moins cher) sur le paquet pré-assemblé → les 5 attrapés ou approchés,
   0 issue de style, `failure_scenario` partout, ancrage 100 %.
2. **Scan pré-vol** : `examples/code-secret.patch` (faux `AKIA…` + faux
   bloc private key) → STOP attendu avant tout envoi, lignes affichées.
3. **Bout en bout** : `/devil-code HEAD~1 glm` sur ce repo → rapport complet,
   correction guidée refusée ou proposée selon le mode.
4. **Chemin d'erreur** : modèle bidon → enveloppe `status:"error"` avec
   detail diagnostique.
5. **Garde-fou swarm** : sur la fixture (l'injection critical/security),
   vérifier qu'un verdict VALABLE est impossible.
6. **Inventaire** : `claude --plugin-dir . plugin details devil` → 6 skills +
   4 agents.
7. **Dogfood méta** : `/devil-code` sur la branche d'implémentation de
   devil-code elle-même, avant merge.
8. Bump 0.3.0, marketplace update, push, uninstall + install, smoke via
   plugin installé.
