---
status: proposed
date: 2026-08-18
chantier: plugin-devil-review (erom-devil v0.7.0)
---

# Architecture technique - devil review (plugin erom-devil v0.7.0)

> 2026-08-18. Complète `brainstorming.md` (les décisions produit y font foi).
> Tout ce qui n'est pas redéfini ici suit `skills/code/SKILL.md` et le
> contrat transport v0.2.0 : les agents ne changent pas d'un octet.

## 1. Arborescence cible

```
plugin/agents/{gemini,glm,deepseek,opus,kimi}.md  INCHANGÉS (transport pur)
plugin/scripts/devil-review-mission.md            NOUVEAU : mission porte de merge
plugin/scripts/devil-review-schema.json           NOUVEAU : copie conforme de devil-code-schema.json
plugin/scripts/stacks/nextjs.md                   NOUVEAU : grille de review Next 16 / Prisma 7
plugin/skills/review/SKILL.md                     NOUVEAU : unitaire (devil au choix, gemini défaut)
plugin/skills/review-swarm/SKILL.md               NOUVEAU : tribunal gemini+glm+deepseek + passe Claude
plugin/README.md                                  MàJ : table exercices, rôles code vs review
```

Versioning : `plugin.json` 0.7.0 ; marketplace bump ; uninstall + install.
Le schéma review démarre en copie conforme du schéma code : fichier séparé
par contrat d'architecture (un schéma par exercice), divergence future libre.
VALIDATE_JQ : la même ligne que `code`, pointée sur `devil-review-schema.json`.

## 2. Contrat transport et INPUTS

Spawn inchangé : `MISSION_FILE` + `SCHEMA_FILE` + `VALIDATE_JQ` + `INPUTS`
(lignes `LABEL:CHEMIN_ABSOLU`). Table des exercices complétée :

| Skill | MISSION_FILE | SCHEMA_FILE | INPUTS |
|---|---|---|---|
| review(-swarm) | devil-review-mission.md | devil-review-schema.json | `DIFF:` (+ `FILES:` + `INTENT:` + `STACK:` optionnels) |

`STACK:` pointe sur le pack du plugin (chemin absolu résolu à l'Étape 0),
jamais copié : les transports embarquent ou lisent les inputs comme
aujourd'hui (agy : `--add-dir` du dossier de chaque input, donc aussi celui
du pack).

## 3. Grammaire des arguments

```
/erom-devil:review [target] [intent.md] [devil] [stack=<nom>|stack=none] [no-gates]
/erom-devil:review-swarm [target] [intent.md] [stack=...] [no-gates]
```

Tokens reconnus et retirés dans cet ordre, le reste = target :

1. `no-gates` (littéral) : saute la porte déterministe, tracé au rapport.
2. `stack=<nom>` : force le pack (`stack=none` = aucun). Nom inconnu
   (aucun `scripts/stacks/<nom>.md`) : STOP avec la liste des packs.
3. dernier argument dans {gemini, glm, deepseek, opus, kimi} : devil
   (unitaire seulement ; défaut gemini).
4. argument `*.md` existant : INTENT explicite (prioritaire sur la chasse).

Résolution du target : table, gardes et cas limites de `code` Étape 1,
inchangés, plus une exclusion : `docs/reviews/` sort du périmètre par
pathspec (`':(exclude)docs/reviews/'` sur les commandes git ; hunks retirés
en mode PR), sinon le rapport d'une review précédente entre dans le diff de
la suivante en mode working tree.

## 4. Étape nouvelle : porte déterministe

Avant tout packaging, seulement si le diff reviewé correspond à l'arbre
courant (mêmes modes que la colonne « Correction guidée : oui » de la table
FILES de `code` Étape 2 ; sinon porte non applicable, mentionnée au rapport) :

1. `package.json` à la racine avec un script `check` : lancer
   `bun run check` (timeout Bash 120000), capturer la sortie ; timeout
   dépassé = rouge, sortie partielle affichée.
2. Pas de script `check` : demander UNE fois la commande de porte à Romain
   (réponse « aucune » = porte sautée, tracée).
3. Vert : la confirmation porte « porte : verte (`bun run check`) » et le
   rapport recopie les dernières lignes brutes de la sortie (jamais un
   résumé : un rapport qui s'auto-compte ment).
4. Rouge : STOP. « Corrige avant de convoquer une review ; les devils ne
   paient pas pour ce que tsc dit gratuitement. » Sortie brute affichée.
   `no-gates` outrepasse ; le rapport porte alors « porte : SAUTÉE sur
   décision » en tête de couverture.
5. Arbre sale en mode branche (porte applicable mais arbre ≠ exactement le
   diff) : une ligne au rapport, pas de machinerie.

La porte est rejouée à chaque re-review (elle coûte < 10 s sur le starter).

## 5. Étape nouvelle : chasse à l'intention

INTENT par priorité, premier trouvé gagne :

1. arg `*.md` explicite ;
2. body de la PR (mode PR, comme `code`) ;
3. `.specs/<branche>/` ou `.specs/<slug proche>/` (specs maison) ;
4. `docs/superpowers/specs/*.md` dont le nom matche la branche ou le sujet ;
5. question à Romain, « aucune » accepté.

Plusieurs candidats aux niveaux 3-4 : les lister, Romain choisit (ou
« aucune »). Sans INTENT : catégorie `intent` interdite au devil (règle
mission existante) et le rapport ouvre sa couverture sur « review sans
intention fournie ». La chasse ne lit que des chemins du repo reviewé ;
aucun contenu n'est envoyé avant le scan anti-fuite (l'INTENT passe au scan
comme aujourd'hui).

## 6. Pack stack

Auto-détection (si pas de `stack=`) : `package.json` racine, `"next"` dans
`dependencies` : pack `nextjs`. Sinon aucun. Le pack sélectionné apparaît
dans la confirmation.

### `scripts/stacks/nextjs.md` (contenu, formulé en grille de review)

En-tête : « Grille de standards LIANTE pour ce repo. Chaque violation est
une issue normale : ancrée file:ligne, failure_scenario concret, sévérité et
catégorie indiquées par la règle. Ne flagge que ce que le DIFF introduit ou
touche. »

Bloc 1 - Autorisation (violation : `critical` / `security`) :
- l'autorisation ne vit JAMAIS dans `middleware.ts` ni `proxy.ts`
  (CVE-2025-29927) ; tout accès aux données passe par la DAL ;
- trois checks par mutation : session, permission, propriété ; l'absence du
  check de propriété est LE défaut à chercher (IDOR) ;
- la propriété vit dans la clause `where`, jamais dans un `if` préalable
  (TOCTOU) ;
- toute Server Action est un endpoint POST public : aucune sans
  `actionClient` (safe-action) ; un fichier ne transite jamais par une
  Server Action (URL présignée).

Bloc 2 - Pièges de version (violation : `high` / `correctness`) :
- Prisma 7 : pas de `url = env(...)` dans le bloc datasource (P1012),
  adapter pg obligatoire, aucune API du moteur Rust ni de Prisma 8 ;
- Zod 4 : `z.email()` / `z.url()`, pas `z.string().email()` ;
- Next 16 : types de route globaux (`LayoutProps`, `PageProps`) ; Pages
  Router, `getServerSideProps`, `getStaticProps`, `next/head` interdits ;
- Node requis au build ET au runtime (Bun installe seulement) ; `next
  start` incompatible `output: "standalone"`.

Bloc 3 - Architecture (violation : `medium` / `architecture`) :
- logique métier en DAL et fonctions pures, pas dans les composants ;
- un seul schéma Zod par formulaire, partagé client/serveur ;
- `next/image` jamais `<img>` ; aucune API Vercel-only (`@vercel/kv`,
  `@vercel/blob`, `@vercel/postgres`) ; toute variable d'env via
  `src/env.ts` ; logs JSON structurés sans PII ni secret.

Bloc 4 - Tests (violation : `medium` à `high` / `tests`) :
- un Server Component async ne se teste pas sous Vitest (Playwright) : un
  test Vitest qui tente = issue ;
- antipatterns bannis : test change-detector (compteur ou snapshot de
  catalogue figé), test qui asserte sur le contenu du code source ;
  comportement seulement ;
- seed déterministe, aucune valeur aléatoire non seedée dans une fixture.

## 7. Mission review (`devil-review-mission.md`)

Base : mission `code` intégrale (périmètre DIFF seul, FILES contexte,
6 critères, anti-bruit, failure_scenario par catégorie, sévérités, verdict
par score), plus :

- **STACK** (nouvel input optionnel) : « grille de standards liante pour ce
  repo ; applique chaque règle au diff, catégorie et sévérité indiquées par
  la règle ; absent, ignore ce paragraphe ».
- **Axe intention promu** (INTENT fourni) : chercher explicitement les trois
  formes : exigence absente ou partielle ; comportement non demandé (scope
  creep) ; exigence implémentée mais fausse. La description de l'issue CITE
  la ligne d'intention concernée.
- **Discipline de sévérité renforcée** : un bug fonctionnel qui produit un
  comportement contraire à l'intention du changement = `high` minimum.
- **Anti-bruit renforcé** : jamais d'issue « vérifie que / assure-toi » ;
  jamais d'explication du code à son auteur ; un même problème en plusieurs
  endroits = UNE issue, autres localisations dans la description ; ne pas
  flagger ce que la porte déterministe applique déjà (types, lint, format).

## 8. `skills/review/SKILL.md` (unitaire)

Frontmatter : `user-invocable: true` ; `allowed-tools: Read, Glob, Grep,
Bash, Agent, AskUserQuestion, Edit, Write` (Write : le rapport). Description
avec triggers : « /erom-devil:review, review de merge, porte de merge,
review finale avant merge ». Première ligne du corps : « Gate de merge :
session au tier pense recommandée (xhigh). `code` reste la critique rapide ;
ici on instruit un dossier. »

Étapes :

1. Étape 0 de `code` + résolution de `STACK_DIR` = `<racine>/scripts/stacks/`.
2. Grammaire § 3, résolution du target par référence à `code` Étape 1
   (+ exclusion `docs/reviews/`).
3. Porte déterministe (§ 4).
4. Chasse à l'intention (§ 5).
5. Packaging par référence à `code` Étape 2 (DIFF, FILES, INTENT) +
   sélection du pack (§ 6).
6. Scan anti-fuite par référence à `code` Étape 3, sur DIFF + FILES +
   INTENT (le pack, fichier du plugin, n'est pas scanné). Jamais sauté.
7. Confirmation enrichie :
   > Mode · cible · N fichiers (±lignes) · FILES ... · porte
   > <verte/rouge STOP/sautée/non applicable> · intent <source/aucune> ·
   > stack <pack/aucun> · scan <clean/forcé> · devil <devil>
   > Je lance la review ?
8. Spawn `erom-devil:<devil>` (contrat § 2, annonce « jusqu'à 9 min »).
9. Ancrage par référence à `code` Étape 6 (déclassement hors-diff).
10. **Passe Claude** (§ 10).
11. **Verdict process** (§ 11) et **rapport persistant** (§ 12), affichage
    du verdict + chemin du rapport dans le chat.
12. Correction guidée par référence à `code` Étape 8 (mêmes modes, mêmes
    règles, low ignorées, le devil ne modifie jamais rien) ; re-review
    max 2, la porte est rejouée, chaque run produit son rapport (suffixe
    `-2`, `-3`).
13. `trash "$TMP_DIR"` en fin de run, succès comme échec.

## 9. `skills/review-swarm/SKILL.md` (tribunal)

Étapes 1 à 7 identiques (un seul TMP_DIR partagé), puis par référence à
`code-swarm` : spawn des 3 devils en UN message (gemini + glm + deepseek,
opus et kimi exclus, décision actée), quorum (3 voix ; 2 voix signalées ;
<= 1 voix = pas de verdict), ancrage PAR VOIX, consolidation par problème de
fond (heuristiques existantes, badges 3/3 2/3 1/3, voix dissonante jamais
écrasée). Ensuite :

- passe Claude (§ 10) UNE fois, sur les groupes consolidés ;
- verdict process (§ 11) sur les groupes vérifiés, la grille
  VALABLE / MODIFICATIONS REQUISES / JETABLE des devils restant affichée
  comme matière ;
- garde-fou sécurité amélioré : une `critical`/`security` ancrée vaut
  NO-GO si Confirmée ou Hypothèse ; si RÉFUTÉE avec preuve, le garde-fou
  tombe et la réfutation figure au rapport (aujourd'hui il ne peut jamais
  tomber) ;
- rapport (§ 12) ; correction guidée ; re-swarm max 1.

## 10. Passe Claude : vérification + frontières

Le cœur du process, exécuté par l'orchestrateur dans la session (tier
pense). Lecture seule stricte : aucune mutation de l'arbre, commandes
lecture-seule et tests unitaires existants uniquement (jamais de migration,
seed ou E2E).

**Périmètre vérifié** : toutes les `critical` et `high` ancrées ; en swarm,
plus les `medium` convergentes (2 voix et +). Les `low` jamais (pas
rentable).

**Par issue** (budget ~2 min, la vérification décisive la moins chère) :

1. lire le code réel au point ancré, tracer ce que le `failure_scenario`
   prétend ;
2. si une commande tranche (grep d'un symbole, test unitaire ciblé
   existant), la lancer et recopier la sortie ;
3. étiqueter :
   - **Confirmée** : la trace ou la sortie prouve le scénario ; preuve
     courte au rapport ;
   - **Réfutée** : preuve du contraire ; annexe du rapport avec la raison,
     jamais supprimée en silence ; exclue du verdict et de la correction
     guidée ;
   - **Hypothèse** : pas tranchable à coût raisonnable ; confiance estimée
     (haute/moyenne/basse) + ce qui la trancherait.

Les issues hors périmètre (medium non convergentes, low) gardent l'étiquette
**Non vérifiée** : elles restent au rapport, jamais promues ni supprimées.

**Balayage frontières** : une passe ciblée de l'orchestrateur sur le diff
entier, quatre angles (les angles morts structurels d'une review scopée au
diff, leçon portail-CCA) :

1. entrée externe qui finit dans un chemin de fichier ou une requête sans
   validation ;
2. réponse HTTP d'erreur qui n'échoue pas en exception (`r.ok` jamais
   testé, status ignoré) ;
3. DX-breakage : variable d'env renommée ou supprimée, port remappé, étape
   de setup nouvelle obligatoire ;
4. copy user-visible qui ne matche plus le comportement.

Findings étiquetés « frontière (Claude) », Confirmés par construction
(preuve : le fichier:ligne lu), soumis aux mêmes règles de verdict.

## 11. Verdict process

Sur les issues ancrées et vérifiées (devils + frontières ; les Réfutées ne
comptent pas) :

| Situation | Verdict |
|---|---|
| >= 1 critical Confirmée | **NO-GO** (corriger avant merge) |
| >= 1 critical Hypothèse | **NO-GO**, levable par décision explicite de Romain, tracée ; catégorie security : non levable tant que ni réfutée ni corrigée |
| >= 1 high Confirmée | **NO-GO**, levable par décision explicite, tracée |
| high Hypothèse ou medium présentes (vérifiées ou non) | **GO AVEC RÉSERVES** (listées) |
| sinon (low seules ou rien) | **GO** |

Une levée de NO-GO s'écrit dans le rapport sous le verdict (date, raison de
Romain) et le frontmatter passe à `NO-GO LEVÉ`. Les verdicts devils (scores,
approve/rework/reject, grille swarm) restent affichés : c'est la matière du
dossier, le verdict process est la décision.

## 12. Rapport persistant

`docs/reviews/YYYY-MM-DD-<slug>.md` dans le repo reviewé (slug = branche ou
target normalisé ; collision même jour, re-review comprise : suffixe `-2`,
`-3`). Écrit par
l'orchestrateur via Write (les externes ne produisent jamais d'artefacts) ;
le commit rejoint le flux de merge, la skill ne committe pas. En français.

```markdown
---
date: <YYYY-MM-DD>
target: <mode et cible>
range: <base_sha>..<head_sha>
devils: <devil (modèle)> | <les 3 en swarm>
porte: <commande + verte/rouge corrigée/sautée/non applicable>
intent: <source | aucune>
stack: <pack | aucun>
verdict: <GO | GO AVEC RÉSERVES | NO-GO>
---

# Review de merge - <cible>

## Verdict : <VERDICT>
<3 lignes : l'essentiel du dossier et la raison du verdict.>

## Couverture
<fichiers et lignes reviewés ; FILES complet/tronqué/omis ; ce qui n'a PAS
été couvert : E2E non relancés, exclusions scan, périmètre. Review sans
intention fournie le cas échéant.>

## Findings
| Statut | Sév | Cat | Fichier | Problème | Scénario | Preuve / à trancher |
<Confirmées (sév décroissante), puis Hypothèses (confiance décroissante),
puis Non vérifiées (sév décroissante) ; badges de convergence en swarm,
« frontière (Claude) » étiquetées.>

## Annexes
### Réfutées <issue + preuve de réfutation>
### Non ancrées <comme aujourd'hui, avec devil d'origine>
### Voix dissonantes <swarm : qui, sur quoi, l'argument>
### Scores devils <grille par devil + moyenne indicative>
### Porte déterministe <dernières lignes brutes de la sortie>
```

## 13. Hors plugin (même chantier, autres repos)

- **web-stack-starter** : `trash .claude/skills/code-review/` (copie Matt
  non adaptée) ; CLAUDE.md, section Vérification, une ligne : « Avant merge
  vers main : `/erom-devil:review-swarm main` (tribunal + vérification +
  rapport dans `docs/reviews/`) ».
- **CLAUDE.md global** : proposer à Romain (jamais d'office) que la ligne
  passe-sécu ajoute `review-swarm` comme couverture du gate de merge.

## 14. Vérification (pas de suite de tests classique)

1. **Fixtures existantes réutilisées** (`examples/code-diff.patch` +
   `code-files.txt`, 5 défauts connus) via `review` complet : porte
   « aucune » (fixture sans package.json), 5 défauts retrouvés, passe
   Claude : les 5 attendus Confirmés avec preuve, ancrage 100 %.
2. **Porte** : sur un repo jetable avec `check` cassé (typo TS), STOP
   attendu avant tout appel modèle ; `no-gates` passe et le rapport le
   trace.
3. **Chasse à l'intention** : repo jetable avec `.specs/<branche>/` :
   détection attendue ; deux candidats : question attendue.
4. **Pack** : package.json avec `next` : sélection nextjs ; `stack=none`
   la débraye ; violation DAL plantée dans une fixture : issue
   critical/security attendue et garde-fou testé en swarm.
5. **Scan pré-vol** : `examples/code-secret.patch` : STOP avant envoi,
   inchangé.
6. **Réfutation** : vérifier sur dogfood qu'au moins une issue devil se
   fait réfuter avec preuve pendant le chantier (le taux historique des
   devils garantit l'occasion) ; sinon fixture dédiée en fin de chantier.
7. **Rapport** : fichier créé au bon chemin, frontmatter complet, sortie
   de porte recopiée brute, pathspec d'exclusion vérifié au run suivant.
8. **Dogfood méta** : `/erom-devil:review-swarm main` sur la branche
   d'implémentation de review elle-même, avant son merge. Le premier
   rapport de `docs/reviews/` du repo devil sera le sien.
9. README + `_memory_` (architecture, key-files, patterns), bump 0.7.0,
   marketplace, uninstall + install, smoke via plugin installé.
