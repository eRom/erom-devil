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
/erom-devil:review main no-gates       # une porte rouge n'arrête plus (tracé)
```

Tokens reconnus et retirés dans cet ordre, le reste = target :
1. `no-gates` (littéral) : la porte tourne quand même si elle est
   applicable, mais un rouge n'arrête plus le run. Tracé au rapport.
2. `stack=<nom>` : force le pack. `stack=none` est reconnu littéralement et
   signifie « aucun pack », sans lecture de fichier. Tout AUTRE nom sans
   fichier `scripts/stacks/<nom>.md` correspondant : STOP avec la liste des
   packs.
3. dernier argument dans {gemini, glm, deepseek, opus, kimi} : devil
   (défaut `gemini`).
4. argument `*.md` existant : INTENT explicite (court-circuite la chasse).

## Étape 0 - Chemins du plugin

Racine du plugin : deux niveaux au-dessus du « Base directory for this
skill » injecté ci-dessus. Résous en absolu, vérifie l'existence
(`MISSION_FILE` et `SCHEMA_FILE` par Read, `STACK_DIR` par Glob
`<STACK_DIR>/*.md`), sinon STOP « plugin corrompu » :
- `MISSION_FILE` = `<racine>/scripts/devil-review-mission.md`
- `SCHEMA_FILE`  = `<racine>/scripts/devil-review-schema.json`
- `STACK_DIR`    = `<racine>/scripts/stacks/`

## Étape 1 - Résolution du target

Identique à `skills/code/SKILL.md` Étape 1 (mêmes formes, mêmes gardes et
cas limites, mêmes règles pour les fichiers non suivis), avec UNE exclusion
en plus : `docs/reviews/` sort du périmètre. Ajoute
`':(exclude)docs/reviews/'` au pathspec des commandes git de diff ; en mode
PR, retire du texte du diff les sections `diff --git` de ces chemins ; et
écarte `docs/reviews/` de la collecte des fichiers non suivis (le même
pathspec fonctionne sur `git ls-files --others`). Sans ces trois gestes,
le rapport d'une review
précédente, jamais commité, entre dans le diff de la suivante.

La règle INTENT de `code` Étape 1 (« pas d'auto-detect `.specs/` ») ne
s'applique PAS ici : l'Étape 3 la remplace par une chasse explicite.

## Étape 2 - Porte déterministe

Applicable si l'arbre courant porte le code du diff reviewé : modes working
tree, branche vs base, range courant (`b` = HEAD) et PR checkoutée, arbre
propre ou sale. Non applicable sur un range historique (`b` différent de
HEAD) ou une PR non checkoutée : porte « non applicable (le diff reviewé
n'est pas dans l'arbre courant) », mentionnée au rapport, continue.

1. `package.json` à la racine avec un script `check` : lance
   `bun run check` (timeout Bash 120000) et capture la sortie. Timeout
   dépassé = rouge, sortie partielle affichée.
2. Pas de script `check` : demande UNE fois la commande de porte à Romain
   (« aucune » accepté = porte sautée, tracée).
3. Verte : continue. Conserve les ~15 dernières lignes brutes de la sortie
   pour le rapport (recopiées, jamais résumées).
4. Rouge : STOP. « Porte rouge : corrige avant de convoquer une review ;
   les devils ne paient pas pour ce que tsc dit gratuitement. » Sortie
   brute affichée. Avec `no-gates` le run continue, et le rapport porte
   « porte : ROUGE, outrepassée sur décision » en tête de couverture, la
   sortie brute en annexe : une porte rouge ne se maquille jamais en porte
   sautée.
5. Arbre sale en mode branche, range courant ou PR checkoutée (porte
   applicable mais l'arbre n'est pas exactement le diff reviewé) : une
   ligne au rapport, rien de plus.

## Étape 3 - Chasse à l'intention

INTENT par priorité, premier trouvé gagne :
1. arg `*.md` explicite ;
2. body de la PR (mode PR), écrit par redirection shell dans un fichier
   `mktemp` qui lui est propre (`TMP_DIR` n'existe qu'à l'Étape 4) ;
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
`TMP_DIR=$(mktemp -d "${TMPDIR:-/tmp}/review-XXXXXX")`. Les fichiers du
paquet s'écrivent par redirection shell, jamais par l'outil Write : un
INTENT citant du texte à tiret cadratin serait refusé.

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
> - Fichiers : <N> (±<lignes>) [· dont <U> non suivis] · FILES <complet / tronqué : n exclus / omis>
> - Porte : <verte (`bun run check`) / rouge outrepassée (no-gates) /
>   sautée (aucune commande) / non applicable>
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
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.score|type==\"number\" and .>=0 and .<=100) and (.verdict|IN(\"approve\",\"rework\",\"reject\")) and (.criteria|has(\"correctness\") and has(\"architecture\") and has(\"security\") and has(\"performance\") and has(\"tests\") and has(\"maintainability\")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type==\"object\" and has(\"score\") and has(\"comment\") and (.score|type==\"number\" and .>=0 and .<=100))) and (.issues|type==\"array\" and all(has(\"severity\") and has(\"category\") and has(\"file\") and has(\"description\") and has(\"failure_scenario\") and has(\"suggestion\") and (.severity|IN(\"critical\",\"high\",\"medium\",\"low\")) and (.category|IN(\"correctness\",\"architecture\",\"security\",\"performance\",\"tests\",\"maintainability\",\"intent\"))))\nINPUTS:\nDIFF:<abs diff>\nFILES:<abs files>\nINTENT:<abs intent>\nSTACK:<abs stack>\n\nExécute la procédure de transport."
)
```

Enveloppe `error` : gabarit d'échec de `code` Étape 6, dont tu réécris la
dernière ligne pour cette skill : « Relance
(`/erom-devil:review <target> <devil>`), autre devil, ou review manuelle. »
Puis FIN DE RUN : pas de passe de vérification, pas de verdict, AUCUN
rapport écrit dans `docs/reviews/`, et `trash "$TMP_DIR"`. Une porte de
merge sans voix de devil ne certifie rien : elle ne laisse jamais un GO
derrière elle.

## Étape 8 - Ancrage

Identique à `code` Étape 6 : extraction des plages modifiées par fichier
depuis le DIFF, issue DÉCLASSÉE si fichier hors périmètre ou ligne hors
plages (tolérance ±3 ; fichier supprimé : ancrage au fichier seul). Les
DÉCLASSÉES vont en annexe « Non ancrées », jamais supprimées en silence,
exclues de la vérification, du verdict et de la correction guidée.

Exception, avant de passer à la suite : toute DÉCLASSÉE de sévérité
`critical` ou `high` est relue une fois. Si son problème de fond vit bien
dans le diff et que seule l'ancre est fausse (le devil a cité la fonction
appelée au lieu du site d'appel), RÉ-ANCRE-la sur la bonne ligne et remets-la
au périmètre. Sinon elle reste en annexe. Une faille réelle ne doit pas
sortir du verdict sur une erreur de pointage.

## Étape 9 - Passe de vérification (ton travail, pas celui du devil)

Lecture seule stricte : aucune mutation de l'arbre, commandes en lecture et
tests unitaires existants uniquement (jamais de migration, de seed ni
d'E2E).

Périmètre : toutes les issues `critical` et `high` ancrées. Budget
~2 minutes par issue : la vérification décisive la moins chère.

SOURCE DE LECTURE, même condition que la porte de l'Étape 2. Modes working
tree, branche vs base, range courant et PR checkoutée : l'arbre courant
porte le code reviewé, tu lis les fichiers directement. Range historique
(`b` différent de HEAD) : lis par `git show b:chemin`, jamais le checkout.
PR NON checkoutée : tu n'as PAS l'état final des fichiers, seulement le
DIFF. Dans ce mode, l'étiquette **Réfutée** est INTERDITE : une issue non
confirmée par le diff seul reste **Hypothèse**, et le garde-fou sécurité ne
peut donc pas tomber. Un fichier lu au mauvais arbre ressemble à une preuve
et n'en est pas.

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
construction (tu l'as lue, cite file:ligne), et une sévérité que tu
attribues avec la MÊME grille que les devils, celle de la mission
(`scripts/devil-review-mission.md`, section « Sévérité de chaque issue »).
Sans sévérité, un finding frontière ne peut pas entrer dans la table de
l'Étape 10. Mêmes règles de verdict ensuite.

## Étape 10 - Verdict

Sur les issues ancrées et vérifiées (devil + frontières ; Réfutées
exclues) :

Première ligne qui matche gagne : la table se lit de haut en bas et
s'arrête à la première situation vraie.

| Situation | Verdict |
|---|---|
| >= 1 critical Confirmée | **NO-GO** non levable (corriger avant merge) |
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
re-review comprise : suffixe `-2`, `-3`). Avant d'écrire : dans tout
matériau recopié (issues des devils, sortie de la porte, body de PR),
remplace chaque tiret cadratin par un tiret simple ; le hook du dépôt
refuse un Write de `.md` qui en contient. La substitution est mentionnée en
une ligne dans la section Couverture. Elle ne touche que ce caractère : le
reste du matériau reste recopié, jamais résumé. En français. Ne committe
pas : le commit rejoint le flux de merge. Structure exacte :

```markdown
---
date: <YYYY-MM-DD>
target: <mode et cible>
range: <base_sha>..<head_sha>
devils: <devil (modèle)>
porte: <commande réellement lancée : verte | rouge outrepassée (no-gates) | aucune commande | non applicable>
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
échéant ; « porte : ROUGE, outrepassée sur décision » le cas échéant ;
la mention de substitution des tirets cadratins le cas échéant.>

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
porte est rejouée, chaque run produit son rapport suffixé. En mode branche,
range ou PR, les corrections doivent être COMMITÉES avant la re-review :
sinon le diff reviewé est inchangé et le devil rendra mot pour mot les
mêmes issues. En mode working tree, elles sont prises telles quelles.

## Règles

- Le devil ne modifie JAMAIS rien ; le rapport est écrit par toi.
- La passe de vérification ne mute JAMAIS l'arbre.
- Scan anti-fuite obligatoire, jamais sauté.
- Tout fichier écrit par cette skill AVANT le rapport final (paquet, body
  de PR, temporaires) s'écrit par redirection shell, jamais par l'outil
  Write : le hook du dépôt refuse un `.md` ou un `.txt` portant un tiret
  cadratin, et les entrées en portent souvent.
- `trash "$TMP_DIR"` en fin de run (succès comme échec).
- Un NO-GO levé se trace dans le rapport, jamais dans le seul chat.
- C'est une porte de merge, pas un lint : pas sur un diff de deux lignes
  (pour ça : `/erom-devil:code`).
