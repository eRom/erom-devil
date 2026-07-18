# Brainstorming — devil code (plugin devil v0.3.0)

> Validé le 2026-07-18. Décisions issues du Q&A Romain × Claude.

## Problème

La chaîne de fabrication est contrôlée en amont : devil-brain interroge le
brainstorming, devil-spec juge les specs contre lui. Mais le maillon final —
le code produit — n'a pas d'avocat du diable. `/code-review` (built-in Claude)
existe, mais c'est le même écosystème qui se relit : l'implémenteur et le
reviewer partagent les mêmes biais. La thèse fondatrice du plugin (diversité
des regards externes) s'applique au code comme aux documents.

## Objectif

`/devil-code` et `/devil-code-swarm` : faire juger un **changement** de code
(jamais un stock) par un ou trois devils externes, avec un verdict actionnable
en gate de commit, de merge ou de PR.

## Matière analysée (sources du design)

- **`/code-review` built-in** (prompts extraits du binaire claude 2.1.214) :
  cibles de diff, pipeline par effort, doctrine anti-bruit du mode low
  (bugs runtime visibles dans le hunk uniquement), `failure_scenario`
  obligatoire par finding.
- **`/review` built-in** : mode PR via `gh pr view --json` + `gh pr diff`,
  le diff PR comme seul scope.
- **`/security-review` built-in** : ne flagger qu'à confiance élevée
  d'exploitabilité réelle, exclusions explicites.
- **`superpowers:requesting-code-review`** : reviewer contre l'INTENTION
  (plan/requirements), pas seulement le code ; review read-only.
- **Ancienne skill perso `devil-code`** (v0 Antigravity) : les 6 critères
  code éprouvés, le flow rapport → correction guidée.

## Contrainte structurante

Le run devil est **hermétique** (contrat transport v0.2.0) : aucun outil, les
contenus sont embarqués dans le prompt entre marqueurs. Le devil ne navigue
pas le repo : tout ce qu'il juge est packagé par la skill orchestratrice en
INPUTS étiquetés. La qualité de la review dépend du paquet.

## Décisions validées

| Sujet | Décision |
|---|---|
| Architecture | **Pattern v0.2.0 pur** : 1 mission + 1 schéma + 2 skills, agents transport inchangés |
| Entrées du devil | **DIFF + FILES + INTENT** (intent optionnel) |
| Sortie | Score 0-100 + verdict approve/rework/reject + critères scorés + issues, ossature devil-spec |
| Anti-bruit | Chaque issue exige `file:ligne` ET un `failure_scenario` concret (entrées → résultat faux) |
| Modes de résolution | PR GitHub, branche vs base, entre 2 commits, working tree non commité (les 4 retenus) |
| Critères scorés | correctness, architecture, security, performance, tests, maintainability |
| Catégories d'issues | les 6 critères + `intent` (dérive vs intention, seulement si INTENT fourni) |
| Swarm | Décalque devil-spec-swarm : gemini + glm + deepseek, quorum ≥ 2, VALABLE / MODIFICATIONS REQUISES / JETABLE |
| devil-opus | **Exclu du swarm** (volontaire, acté), disponible en unitaire (`/devil-code opus`) |

## Résolution du target

Dernier argument = devil (`gemini` défaut, `glm`, `deepseek`, `opus`), comme
`/devil-spec`. Le reste :

| Argument | Mode | Résolution |
|---|---|---|
| `123` (nombre pur) | PR | `gh pr view --json title,body,baseRefName,…` + `gh pr diff` |
| `a..b` (contient `..`) | range | `git diff a..b` |
| ref valide (`main`, `HEAD~1`, sha) | branche vs base | `git diff ref...HEAD` |
| *(rien)*, tree sale | working tree | `git diff HEAD` (staged + unstaged) |
| *(rien)*, tree propre | branche vs base auto | base = HEAD branch d'origin, fallback main/master |

INTENT, par priorité : un path `.md` passé en argument → le body de la PR en
mode PR → rien. Pas d'auto-detect `.specs/` : trop de risque d'embarquer une
spec sans rapport avec le diff. Confirmation systématique avant lancement
(mode, cible, N fichiers, ±lignes, devil, intent).

## Packaging hermétique

Fichiers temp dans un `mktemp -d`, passés en `INPUTS:` étiquetés :

- **DIFF** — le diff unifié brut. Jamais tronqué ; au-delà de ~1 Mo, refus et
  suggestion de découper la review.
- **FILES** — état final de chaque fichier modifié, chacun sous un en-tête
  `=== FILE: chemin ===`. Source : checkout local, ou `git show sha:path`
  pour un range historique. Budget ~200 Ko : au-delà, garder les fichiers
  les plus touchés (par lignes de diff) et lister les exclus en tête du
  paquet. Fichiers supprimés mentionnés, binaires exclus et mentionnés.
- **INTENT** *(optionnel)* — le doc d'intention.

Cas dégradé : PR non checkoutée localement → FILES omis, signalé dans la
confirmation et le rapport.

## Mission (le nerf de la guerre : l'anti-bruit)

Reviewer senior avocat du diable sur un **changement**. Règles dures :

- Le DIFF est le seul périmètre des issues ; FILES n'est que du contexte.
  Interdit de flagger du code pré-existant non touché.
- Chaque issue : `file:ligne` + `failure_scenario` concret + suggestion
  actionnable. Pas de scénario d'échec crédible = pas d'issue.
- Style et naming purs interdits. Confiance élevée exigée (héritage
  `/security-review`).
- Si INTENT fourni : juger aussi l'alignement du code à l'intention
  (catégorie `intent`), l'angle de `requesting-code-review`.
- Verdict : ≥ 80 approve, 50-79 rework, < 50 reject. Reject = le changement
  est structurellement mauvais, réécrire coûte moins cher que corriger.

## Rapport, correction, swarm

- Rapport unitaire : gabarit `/devil-spec` + colonne Fichier dans le tableau
  d'issues.
- Correction guidée (Edit, sévérité décroissante, `low` ignorées, max 2
  re-review, le devil ne modifie jamais rien) **uniquement si le diff reviewé
  correspond au working tree actuel**. PR non checkoutée ou range historique
  → rapport seul.
- Swarm : 3 voix en parallèle, quorum ≥ 2, consolidation par problème de
  fond (aidée par file:line), badges 3/3 2/3 1/3, voix dissonante jamais
  écrasée, max 1 re-swarm.

## Non-buts

- Pas de sous-agent contexteur enrichissant le paquet (callers, conventions) :
  piste v0.4 si le dogfood montre des faux positifs par manque de contexte.
- Pas de devils outillés sur le repo : casserait le contrat transport pur et
  exposerait le repo à des modèles tiers outillés. Écarté définitivement.
- Pas de review de stock (dossier/fichier sans notion de changement) : le
  devil juge un changement.
- Pas de critère scoré conditionnel `intent` : le schéma reste stable, la
  dérive d'intention vit dans les catégories d'issues.
- Pas de `--fix` automatique : la correction reste pilotée par l'orchestrateur
  avec validation de Romain, comme devil-spec.
- Corollaire v0.1.0 inchangé : les entrées transitent par des fournisseurs
  externes, jamais de secrets dans un diff soumis aux devils.

## Points d'attention livraison

- Collision de nom réglée : la skill perso `~/.claude/skills/devil-code`
  (v0 Antigravity) a été trashée le 2026-07-18.
- MàJ plugin = uninstall + install (gotcha connu).
- Bump manifest 0.3.0 ; README et `_memory_` à mettre à jour à
  l'implémentation.
