# Rétrospective - chantier plugin devil (v0.1.0 + v0.2.0)

Date de la rétro : 2026-07-18. Périmètre : tout le projet (naissance → HEAD `ec50146`).
Toutes les heures de ce rapport sont en **CEST** (les digests de sessions horodatent en UTC,
décalage +2h réaligné en synthèse, voir caveats).

## 1. Fiche du chantier

| Mesure | Valeur |
|---|---|
| Période | 2026-07-18, 10:40 → 18:23 CEST (une seule journée) |
| Sessions | 7 digérées (+1 stub vide `5f2cd87a`, 274 o) |
| Prompts user | 35 |
| Erreurs d'outils | 14 |
| Coupures quota | 0 |
| Tokens output | ~2 268 989 (dont 1 578 779 pour la session principale) |
| Subagents | 26 sur la session principale |
| Commits | 29 (11:10 → 18:23), +4616/-434 lignes |
| Livrables | devil v0.1.0 (spec reviewers), v0.2.0 (devil brain, transport pur), + `devil-opus` (commit manuel) |

Structure de la journée : une session-fleuve unique `976fc18b` (10:40 → 17:13, deux chantiers
séparés par un `/compact` piloté à 13:12), 3 sessions teammates devils (14:13-14:27),
1 session plan-reviewer (14:46-15:17), et un commit manuel final hors session (18:23).

## 2. Résumé exécutif

Une journée, deux versions livrées, trajectoire git impeccable : 0 revert/fixup/WIP sur
29 commits, 28/29 en Conventional Commits FR. Les trois patterns gagnants, tous prouvés par
horodatages croisés sessions/git : tribunal de 3 devils en parallèle (3 reviews convergentes
en 15 min), pré-vol adversarial du plan avant Task 1 (1 Important corrigé avant exécution),
boucle subagent impl+review (7 tasks en 54 min, zéro erreur d'outil sur la fenêtre). Les
coûts principaux : un bug de contrat SendMessage répété à l'identique sur 3/3 teammates
(2 relances manuelles), et 4 recadrages de Romain sur des réflexes de prudence non calibrés.
La mémoire projet a été tenue à jour dans le tour du chantier, gotchas promus en fiches le
jour même.

## 3. Ce qui a marché - à répliquer

- **Tribunal 3 devils en fan-out parallèle.** Lancés en un seul message à 13 s d'écart
  (14:13:54, 14:14:01, 14:14:07), 3 reviews structurées et convergentes reçues en ~15 min
  (scores 73/77/83, mêmes 2 angles morts identifiés), amendements committés 4 min après le
  dernier verdict (`f444f4f`, 14:31). Preuve : digests 30f7bfd3/595d2f5f/685cca94 + git.
- **Pré-vol adversarial du plan AVANT Task 1, avec vérifications mécaniques réelles.** Le
  plan-reviewer (session `9f7dffa5`, 14:46-15:17, 0 erreur) a exécuté le sed du plan pour de
  vrai, parsé le JSON du schéma, compté les fences. Verdict GO_WITH_FIXES : 1 Important
  (VALIDATE_JQ dupliqué par référence, risque de désynchronisation silencieuse) corrigé avant
  exécution (`d40597d`, 15:19). Déjà documenté dans `patterns.md` - confirmé une fois de plus.
- **Boucle subagent impl+review en cadence serrée.** 7 tasks livrées et reviewées en 54 min
  (15:20-16:14), paires impl+review de 3-5 min, gap constant de 4-5 min, zéro erreur d'outil
  sur la fenêtre (6 des 8 erreurs de la session sont hors de cette fenêtre). Côté git :
  10 commits atomiques en 58 min, aucun fourre-tout. Ledger complet dans `.superpowers/sdd/`
  (7 briefs + 7 reports + 7 diffs). Pattern déjà dans le playbook (pipeline spec→plan→TDD→
  double review) - confirmé une fois de plus.
- **Boucle courte détection → fix → mémoire.** Gotcha manifest (clé `agents` rejetée) : fixé
  8 min après sa création (`b19f295` 12:21 → `87187bc` 12:29), promu fiche auto-memory
  atomique le jour même. Les 3 gotchas du chantier (ollama cloud, manifest, uninstall+install)
  ont chacun leur fiche avec why/how-to-apply et liens croisés.
- **Smoke + dogfood avant revue finale.** 4 smoke tests (16:20) puis 3 dogfood self (16:27)
  avant la revue finale de branche sur opus (16:47). Le smoke v0.1.0 : les 3 devils détectent
  les 6 défauts plantés de la fixture (scores 15/22/12, reject unanime).
- **Clôtures de phase actées par la mémoire.** 2 mises à jour `_memory_/` dédiées, une par
  fin de phase (`70afb04` 13:11, `e5af5e6` 17:13), réceptacles cohérents avec l'état final
  v0.2.0 vérifié par le lecteur réceptacles.
- **Auto-récupération teammate.** Le devil deepseek (`685cca94`) a corrigé seul son format
  SendMessage (1 prompt, 2 envois), là où les deux autres ont eu besoin d'une relance.

## 4. Ce qui a coûté - frictions chiffrées

- **Contrat SendMessage violé 3/3.** Les 3 teammates devils ont passé `message` comme objet
  JSON au lieu d'une string : erreur `invalid_union … expected string, received object`
  identique mot pour mot dans les 3 digests (14:17, 14:18, 14:20). 2/3 ont nécessité une
  relance explicite du team-lead (« ton enveloppe ne m'est pas parvenue »), allongeant chaque
  session à 11-13 min pour 2 prompts. Le filet « écris le fichier ET renvoie l'enveloppe » a
  rattrapé le coup mais n'a pas empêché la première tentative de rater.
- **4 recadrages de Romain sur des réflexes de prudence non calibrés** (coût en aller-retours,
  pas en travail perdu) : 11:47 pull ollama inutile sur modèles `:cloud` (depuis promu en
  fiche mémoire), 12:18 inférence erronée après un `/model`, 12:49 avertissement
  confidentialité non sollicité sur l'envoi des specs aux devils externes, 13:03 hésitation à
  committer dans `~/.claude`.
- **Import « verbatim » réécrit à 75 min.** SKILL.md créé à 156 lignes (`e27c4f3`, 11:10),
  quasi entièrement réécrit (93/-97) à 12:26 (`b51612e`). L'import initial n'a servi que de
  brouillon.
- **Diagnostic repayé sur 2 agents.** Le bug « lire `api_error_status` + `.result`, pas
  stderr » corrigé en `17621f8` (12:30) sur les agents glm et deepseek créés 4-6 min plus tôt :
  le pattern erroné du premier agent avait été propagé avant d'être vérifié.
- **4 erreurs d'outils évitables** sur 14 : 2× « Unknown skill » (14:13, deux essais de
  nommage `devil-spec-swarm` avant le bon appel d'Agent), 2× « File has not been read yet »
  (17:11, écriture `_memory_/` en fin de session sans Read préalable).
- **Plans committés d'un bloc.** plan.md v0.1.0 +1011 lignes (`dae2e73`) et v0.2.0
  +1030 lignes (`6ee3726`), sans checkpoint intermédiaire : aucune trace git si la rédaction
  avait été interrompue.
- **Friction infra ponctuelle.** Push SSH refusé (publickey) sur ce repo, contourné en
  basculant le remote en HTTPS (le marketplace était déjà en HTTPS).
- **HEAD hors harnais.** `ec50146` « add devil-opus » (18:23) est un commit manuel de Romain
  (aucun trailer Co-Authored-By, aucune session active, après 69,6 min de silence). Pas une
  dérive du harnais, mais conséquence réelle : `devil-opus` n'existe ni dans `_memory_/`
  (figée v0.2.0 à 17:13) ni dans la convention de commits.

## 5. Leçons proposées

**→ Playbook (`~/.claude/erom-playbook.md`)**

- **Pré-vol adversarial du plan avant Task 1.** Avant de lancer la première tâche d'un plan
  subagent-driven, une revue adversariale qui exécute réellement les commandes du plan (sed,
  parse JSON, comptage de fences) plutôt que de le relire attrape ce que la relecture rate :
  1 Important corrigé avant exécution, puis 7 tâches en 54 min sans erreur d'outil
  (erom-agence-devil 2026-07-18, session 9f7dffa5, commit d40597d). Complète la leçon
  pipeline existante : elle porte sur l'AVANT-implémentation, la double review sur l'après.
- **Enveloppe teammate : une string, jamais un objet.** Un teammate qui renvoie du JSON via
  SendMessage doit passer la string JSON stringifiée dans `message` ; 3/3 teammates ont buté
  sur `invalid_union` (objet brut) malgré une consigne d'enveloppe précise, 2/3 ont exigé une
  relance. Le spawn prompt doit dire « message = la string JSON (stringifiée), pas l'objet »
  (erom-agence-devil 2026-07-18, sessions 30f7bfd3/595d2f5f/685cca94).

**→ CLAUDE.md global (diff proposé, section `<tooling>`)**

- Enrichir la ligne rtk existante avec sa limite, absente aujourd'hui : même préfixée
  `command`, une sortie git peut rester biaisée (plage de commits condensée, log plafonné à
  50 lignes) ; pour un état décisionnel, vérifier par `git rev-parse` ciblé ou redirection
  fichier + Read. (Confirmé 2 chantiers de suite : retro-caserne 08/07, erom-agence-devil
  18/07.)

**→ Gotchas projet (`_memory_/gotchas.md`)**

- **Devils externes : pas d'avertissement de confidentialité.** L'envoi des specs/brainstorms
  aux devils (Google, Ollama cloud) est le design même du plugin ; Romain l'a recadré
  explicitement (12:49, « c'est implicite »). Ne pas répéter l'avertissement.

**→ Nettoyages réceptacles (mineurs, optionnels)**

- Fusionner le doublon interne : la phrase « PAS de clé `agents` (rejetée ; auto-découverte) »
  apparaît quasi mot pour mot dans `gotchas.md` ET `key-files.md`.
- Référencer `.superpowers/sdd/` (ledger complet du chantier v0.2.0) dans `key-files.md`, où
  on l'attendrait ; il n'est cité qu'en une ligne de `patterns.md`.

**Déjà capturé, aucune écriture (confirmations)** : ollama cloud sans pull, manifest sans clé
`agents`, update = uninstall+install (3 fiches auto-memory), pipeline subagent-driven
(playbook), gotcha rtk côté projet (gotchas.md).

## 6. Outillage manquant

- **digest.ts : timestamps en heure locale.** Les digests émettent de l'UTC, git du CEST :
  les 2 lecteurs sessions ont livré des timelines décalées de 2h, le lecteur git a failli
  conclure à une « pause déjeuner 12:30-14:09 » qui était du travail actif (tribunal +
  brainstorm). Réalignement manuel nécessaire en synthèse. Fix : convertir en local ou
  étiqueter le fuseau.
- **Skill retro : mission du lecteur Linear à recâbler.** Un subagent `retro-reader` n'a ni
  ToolSearch ni outils MCP : 3 pistes tentées, 0 accès, lot systématiquement vide. Soit
  l'orchestrateur lit Linear lui-même, soit la mission fournit un accès qui marche.
- **Validation du payload SendMessage dans les procédures devils.** 3/3 échecs sur le même
  type : un rappel « stringify » dans les procédures d'agents (ou une normalisation côté
  appelant) éliminerait la classe d'erreur.
- **Script de généralisation des agents non formalisé.** Le sed qui a régénéré devil-deepseek
  (`c10efd5`) n'existe pas en fichier ; si la v0.3 généralise d'autres agents (devil-opus est
  déjà arrivé hors convention), un script commité économise la re-frappe.

## 7. Caveats méthodologiques

- **Fuseau horaire.** Digests en UTC, git en CEST : les rapports sources citent des heures
  UTC, ce rapport a tout réaligné en CEST (+2h). Vérifié par recoupement exact (7 tasks
  15:20-16:14 UTC+2 ↔ commits 15:21-16:19 ; plan-reviewer 14:46-15:17 ↔ fixes pré-vol 15:19).
- **Session `9f7dffa5`** : 58 % du transcript est du base64 exclu du digest ; ses volumes
  (357 k tokens, 31 min) sont non représentatifs. Son verdict GO_WITH_FIXES a été reconstitué
  via le commit `d40597d`, pas via le digest.
- **Session vide `5fdaa786`** (17:03, `/resume`, 10 s, 0 prompt) : cause non déterminée sans
  deep-dive jsonl. Session stub `5f2cd87a` (274 o, 2 lignes de titre, aucun timestamp) :
  exclue par le digest, aucun contenu perdu.
- **Linear injoignable** : la trajectoire des issues (projet team EAT) n'a pas pu être croisée
  avec le travail réel. Signal secondaire non résolu : un serveur `erom-linear` apparaît dans
  l'historique d'usage sans config active (régression ou renommage ?).
- **Non tranché** : le rendement du volume TaskCreate/TaskUpdate (37+72 appels sur 29 prompts)
  n'est pas mesurable depuis les digests ; la référence « Retiens : 1,2,3,4 et 5 - v0.3 : 7,
  9 » (16:45) pointe une liste non visible dans les digests.
- Aucun secret ni credential rencontré dans les lots (les valeurs type AUTH_TOKEN=ollama sont
  des placeholders documentés).

## 8. Écritures appliquées

Validées par Romain le 2026-07-18 (5/6) :

| # | Destination | Statut |
|---|---|---|
| 1 | Playbook - pré-vol adversarial du plan avant Task 1 | **Appliquée** (section Qualité et livraison) |
| 2 | Playbook - enveloppe teammate = string JSON stringifiée | **Appliquée** (section Orchestration multi-agents) |
| 3 | CLAUDE.md global - nuance rtk « même avec command » | **Refusée** |
| 4 | gotchas.md - devils externes sans avertissement confidentialité | **Appliquée** (nouvelle section) |
| 5 | Fusion doublon clé `agents` gotchas.md/key-files.md | **Appliquée** (key-files.md renvoie vers gotchas.md) |
| 6 | key-files.md - référence au ledger `.superpowers/sdd/` | **Appliquée** (nouvelle section Ledger de chantier) |
