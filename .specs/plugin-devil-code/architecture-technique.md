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

Cas limites (tribunal 2026-07-18) :

- `gh` non installée alors qu'un numéro de PR est fourni → stop « gh CLI
  requis pour le mode PR », aucun fallback.
- Arg qui est un chemin existant (`src/index.ts`, `lib/`) → stop explicite :
  le target désigne un changement (PR, range, ref), pas un chemin — la
  review de stock est un non-but du brainstorming. Message : « target =
  PR, range ou ref ; review d'un fichier/dossier hors périmètre ».
- Diff ne contenant QUE des fichiers binaires → stop « aucun contenu texte
  à reviewer », aucun appel modèle.
- `git show b:chemin` en échec sur un range historique (rename compliqué,
  état incohérent) → fichier listé `=== FILE: chemin (indisponible au
  commit b) ===` sans contenu, la review continue.

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
| range `a..b` avec `b` = HEAD **et tree propre** | `cat` du checkout | oui |
| range `a..b` avec `b` = HEAD, tree sale | `cat` du checkout (HEAD via `git show`) | non — le working tree ≠ le diff reviewé |
| range historique (`b` ≠ HEAD) | `git show b:chemin` | non — rapport seul |
| PR checkoutée localement **(tree propre)** | `cat` du checkout | oui |
| PR checkoutée, tree sale | `git show HEAD:chemin` | non — rapport seul |
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

Mécanique du choix « exclure » : le DIFF est régénéré avec un pathspec
d'exclusion (`git diff … -- ':!chemin'` ; en mode PR, hunks du fichier
retirés du diff téléchargé), le fichier est retiré de FILES, l'exclusion est
listée dans la confirmation, puis le scan est rejoué sur le paquet réduit.

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
has("score") and has("verdict") and has("summary") and has("criteria") and has("issues") and (.score|type=="number" and .>=0 and .<=100) and (.verdict|IN("approve","rework","reject")) and (.criteria|has("correctness") and has("architecture") and has("security") and has("performance") and has("tests") and has("maintainability")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type=="object" and has("score") and has("comment") and (.score|type=="number" and .>=0 and .<=100))) and (.issues|type=="array" and all(has("severity") and has("category") and has("file") and has("description") and has("failure_scenario") and has("suggestion") and (.severity|IN("critical","high","medium","low")) and (.category|IN("correctness","architecture","security","performance","tests","maintainability","intent"))))
```

Ligne testée sur jq 1.8.1 (tribunal 2026-07-18) : sample conforme → pass ;
issue sans `failure_scenario` → fail ; critère à 150 ou remplacé par une
string → fail.

### Ancrage vérifié (orchestrateur, dogfood Q3)

Après réception d'une enveloppe `ok`, l'orchestrateur extrait du DIFF les
plages de lignes modifiées PAR FICHIER (parsing des en-têtes de hunks
`@@ -a,b +c,d @@`, côté état final `+c,d`) et vérifie le champ `file` de
chaque issue à deux niveaux (tribunal 2026-07-18, ancrage ligne exigé par
le brainstorm) :

1. fichier hors du périmètre du diff → DÉCLASSÉE ;
2. fichier dans le périmètre mais ligne hors des plages modifiées, avec une
   tolérance de ±3 lignes de contexte → DÉCLASSÉE. Cas particulier :
   fichier supprimé → ancrage au fichier seul (pas de plage à l'état final).

Issue DÉCLASSÉE = affichée dans une section « Non ancrées » sous le tableau
(jamais supprimée en silence), exclue de la correction guidée et du
garde-fou sécurité. En swarm, l'ancrage est appliqué PAR VOIX, avant
consolidation.

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
7. Correction guidée UNIQUEMENT si le mode l'autorise (table § 4).
   Séquence par issue, sévérité décroissante (`low` ignorées sauf demande),
   issues non ancrées exclues :
   1. présenter l'issue (fichier:ligne, problème, scénario, suggestion) ;
   2. Romain tranche : **appliquer** / **ignorer** / **stop** (arrêt de la
      passe, récap immédiat) ;
   3. si appliquer : relire le fichier concerné, puis Edit ;
   4. issue suivante ; à la fin, récap une ligne par issue (appliquée /
      ignorée) puis proposer re-review (max 2, même devil) ou commit.
   Le devil ne modifie jamais rien : c'est l'orchestrateur qui édite.
8. `trash "$TMP_DIR"`.

### `/devil-code-swarm [target] [intent.md]` (tribunal)

1. Résolution + packaging + scan UNE fois (TMP_DIR orchestrateur partagé en
   lecture par les 3 agents ; chaque agent garde son TMP_DIR interne).
2. Spawn des 3 devils (`gemini`, `glm`, `deepseek` — opus exclu, décision
   actée) en UN SEUL message, prompt identique.
3. Quorum v0.2.0 : 3 voix pleines ; 2 voix → rapport ouvert sur la voix
   absente ; ≤ 1 voix → pas de verdict, rapport d'échec.
4. Ancrage par voix, puis consolidation par PROBLÈME DE FOND. Doctrine
   v0.2.0 rappelée : la consolidation est le travail LLM natif de
   l'orchestrateur — AUCUN agent dédié, algorithme ou seuil à coder.
   Heuristiques d'aide au jugement (tribunal 2026-07-18) :
   - candidats à l'équivalence : même fichier ET plages de lignes
     chevauchantes ou voisines (±10) ET même problème de fond (la catégorie
     aide mais ne décide pas — un même bug peut être classé correctness par
     l'un, security par l'autre) ;
   - deux issues au même endroit peuvent rester deux problèmes distincts ;
     en cas de doute, NE PAS fusionner (deux entrées valent mieux qu'un
     amalgame) ;
   - badge = nombre de voix du groupe ; sévérité du groupe = la plus haute ;
     suggestion la plus actionnable conservée.
5. **Verdict** (dogfood Q1 — 3/3) :

| Situation (voix exprimées) | Verdict |
|---|---|
| ≥ 2 approve ET 0 reject | **VALABLE** |
| ≥ 2 reject | **JETABLE** |
| tout le reste (dont split approve/rework/reject) | **MODIFICATIONS REQUISES** |

   **Garde-fou sécurité** : ≥ 1 issue `critical` de catégorie `security`
   ANCRÉE portée par au moins un devil → le verdict ne peut pas être
   VALABLE : un VALABLE issu de la table est ramené à MODIFICATIONS
   REQUISES, avec mention explicite du garde-fou dans le rapport ; un
   JETABLE reste JETABLE (le garde-fou ne remonte jamais un verdict,
   tribunal 2026-07-18). Pas de score consolidé : grille des scores par
   devil + moyenne indicative.
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
8. MàJ README (sections `/devil-code` et `/devil-code-swarm`) + `_memory_`
   (architecture, key-files, patterns : exercice code, scan pré-vol,
   ancrage, garde-fou swarm).
9. Bump 0.3.0, marketplace update, push, uninstall + install, smoke via
   plugin installé.
