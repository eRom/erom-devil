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
> 9 min).
> - Cible : <mode/cible>
> - Fichiers : <N> (±<lignes>) [· dont <U> non suivis] · FILES <complet / tronqué : n exclus / omis>
> - Porte : <état> · Intent : <source> · Stack : <pack>
> - Scan : <clean / forcé> · Correction guidée : <disponible / rapport seul>
>
> Je lance ?

Pas d'argument devil : le tribunal est fixe. Un nom de devil passé quand
même (`review-swarm main opus`) n'est PAS traité comme un target : STOP avec
« le tribunal est fixe (gemini + glm + deepseek) ; pour un second avis
indépendant : `/erom-devil:review <target> opus` ». `stack=`, `no-gates` et
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
  1 voix ou moins : FIN DE RUN, comme la garde de terminaison de
  `/erom-devil:review` Étape 7. Pas de verdict, AUCUN fichier écrit dans
  `docs/reviews/`, `trash "$TMP_DIR"`, et un récap d'échec affiché dans le
  chat seulement, qui propose une relance ou un passage en unitaire. Un
  tribunal à une voix ne certifie rien : il ne laisse pas de rapport
  derrière lui ;
- une enveloppe `status: "error"` compte comme UNE VOIX ABSENTE, pas comme
  une fin de run : au tribunal, le quorum ci-dessus remplace la garde de
  terminaison de `/erom-devil:review` Étape 7, qui ne vaut que pour un
  devil unique. Deux voix pleines et une en erreur donnent un run à 2 voix,
  pas un run mort ;
- un retour qui n'est pas une enveloppe JSON valide du tout compte lui
  aussi comme voix absente (ne JAMAIS interpréter un texte d'erreur comme
  une review) ;
- ancrage PAR VOIX (les DÉCLASSÉES sortent avant consolidation, listées en
  « Non ancrées » avec leur devil), AVEC l'exception de ré-ancre de
  `/erom-devil:review` Étape 8 et ses deux conditions : toute DÉCLASSÉE
  `critical` ou `high` est relue une fois, et si son problème de fond vit
  dans le diff alors que seule l'ancre est fausse, elle est ré-ancrée et
  rejoint la consolidation. Une faille réelle ne doit pas sortir du verdict
  sur une erreur de pointage, pas plus à trois voix qu'à une. À trois voix,
  le test de doublon se fait sur les issues ancrées de TOUTES les voix, pas
  seulement celle du devil qui a produit la DÉCLASSÉE : sans quoi un même
  problème mal ancré par les trois entre trois fois ;
- consolidation par PROBLÈME DE FOND (mêmes heuristiques : équivalence par
  fichier + plages voisines à `±10` + même fond ; en cas de doute NE PAS
  fusionner ; badges de convergence SUR LES VOIX EXPRIMÉES, donc 3/3, 2/3
  et 1/3 à trois voix, 2/2 et 1/2 à deux ; sévérité du groupe = la plus
  haute ; suggestion la plus actionnable conservée ; tri par convergence
  puis sévérité).

## Étape 9 - Passe de vérification

Identique à `/erom-devil:review` Étape 9 (vérification, grille de
réfutation, balayage frontières), appliquée UNE fois aux groupes
consolidés. Périmètre élargi : `critical` et `high`, tout groupe de
catégorie `security` quelle que soit sa sévérité, plus les `medium`
convergentes (2 voix et plus) des autres catégories.

## Étape 10 - Verdict

D'abord la matière devils : grille VALABLE / MODIFICATIONS REQUISES /
JETABLE de `code-swarm` Étape 8, scores par devil + moyenne indicative,
voix dissonante jamais écrasée (reject isolé, écart > 30 sur un critère :
qui, sur quoi, son argument).

Puis le verdict process : la table de `/erom-devil:review` Étape 10,
appliquée aux groupes vérifiés.

Garde-fou sécurité EN DERNIER, version vérifiée : une issue de sévérité
`critical` ET de catégorie `security`, ANCRÉE, vaut NO-GO si Confirmée ou
Hypothèse. Il s'applique APRÈS la table et ne peut que DURCIR le verdict,
jamais l'adoucir : un NO-GO reste NO-GO, un GO ou un GO AVEC RÉSERVES
devient NO-GO. Si l'issue est RÉFUTÉE, le garde-fou tombe et la réfutation
figure en annexe du rapport ; mais une réfutation qui fait tomber CE
garde-fou doit citer une sortie de commande recopiée ou un file:ligne
réellement lu DANS L'ARBRE QUI PORTE LE CODE REVIEWÉ (Étape 9), jamais un
raisonnement seul et jamais un fichier lu au mauvais arbre. En mode PR non
checkoutée, aucune réfutation n'est recevable : le garde-fou ne tombe pas.
C'est la seule chose qui puisse lever un blocage sécurité, et elle est
produite par un juge unique : elle se paie en preuve.

## Étape 11 - Rapport, correction, clôture

Rapport : même gabarit que `/erom-devil:review` Étape 11, enrichi swarm :
- frontmatter `devils: gemini + glm + deepseek` (voix absente notée) ;
- colonne `Conv` (badge de convergence) dans le tableau Findings ; un
  finding frontière, qui n'a pas de voix devil, y porte `Claude` ;
- annexe « Voix dissonantes » ;
- annexe « Scores devils » : grille par devil + moyenne indicative, ET le
  commentaire de chaque devil sur chaque critère (18 lignes courtes) : les
  scores seuls ne s'expliquent pas. Elle REMPLACE l'annexe « Critères du
  devil » du gabarit unitaire, qui n'a pas de sens à trois voix.

Affichage chat : verdict, comptes par statut, chemin du rapport.

Correction guidée : mêmes règles que `/erom-devil:review` Étape 12, ordre
de passage : groupes convergents (unanimes puis majoritaires sur les voix
exprimées) Confirmés d'abord, puis
critical / high isolées Confirmées, puis Hypothèses hautes. Re-swarm max 1,
porte rejouée, rapport suffixé. `trash "$TMP_DIR"` en fin de run.

## Règles

- Toutes celles de `/erom-devil:review`, plus :
- opus et kimi ne siègent pas au tribunal ; second avis indépendant en
  unitaire : `/erom-devil:review <target> opus`.
- Coût : 3 modèles en parallèle + une passe de vérification. C'est LA porte
  avant merge to main ; pour une critique en cours de route :
  `/erom-devil:code(-swarm)`.
