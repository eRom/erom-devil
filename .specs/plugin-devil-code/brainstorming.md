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
confirmation et le rapport. Le verdict rendu en DIFF seul n'est pas bridé
mais porte un badge d'ouverture « REVIEW SUR DIFF SEUL — contexte réduit »
(dogfood Q10), et la mission demande au devil de calibrer : issue dépendant
du contexte absent → sévérité prudente, dépendance nommée.

**Scan pré-vol anti-fuite (dogfood Q2, 3/3)** — avant tout envoi :

- Exclusion dure du paquet FILES : fichiers type `.env*`, `*.pem`, `*.key`,
  `credentials*`, `secrets*` (liste figée dans la skill). Exclusion signalée.
- Scan de DIFF + FILES par une douzaine de regex à haute valeur (blocs
  private key, `AKIA…`, `ghp_…`, `sk-…`, `xox…`, affectations
  password/token/secret/api_key). Un hit → STOP avant envoi, lignes
  suspectes affichées, Romain tranche : exclure le fichier, annuler, ou
  forcer en connaissance de cause. Zéro dépendance nouvelle ; le biais
  assumé est le faux positif (un STOP à tort coûte une relecture, un faux
  négatif coûte une fuite irréversible).

## Mission (le nerf de la guerre : l'anti-bruit)

Reviewer senior avocat du diable sur un **changement**. Règles dures :

- Le DIFF est le seul périmètre des issues ; FILES n'est que du contexte.
  Interdit de flagger du code pré-existant non touché.
- Chaque issue : `file:ligne` + `failure_scenario` concret + suggestion
  actionnable. Pas de scénario d'échec crédible = pas d'issue.
- Sémantique du `failure_scenario` (dogfood Q4) : « conséquence concrète et
  située », déclinée par catégorie — correctness/security/performance/tests
  = entrées → résultat faux ; architecture/maintainability = le coût futur
  nommé et son déclencheur (« ajouter un 4e transport forcera à dupliquer X
  dans 3 skills »). « C'est moche » interdit ; « ça coûtera X au prochain
  Y » exigé. Un seul champ, schéma inchangé, la mission porte la nuance.
- Style et naming purs interdits. Confiance élevée exigée (héritage
  `/security-review`).
- Si INTENT fourni : juger aussi l'alignement du code à l'intention
  (catégorie `intent`), l'angle de `requesting-code-review`.
- Verdict : ≥ 80 approve, 50-79 rework, < 50 reject. Reject = le changement
  est structurellement mauvais, réécrire coûte moins cher que corriger.

**Ancrage vérifié (dogfood Q3)** — le contrat transport (VALIDATE_JQ + retry
+ SCHEMA_INVALID) couvre la syntaxe, pas les faits. En plus :

- VALIDATE_JQ renforcé : bornes de score 0-100, enums severity/category.
- Au rapport, l'orchestrateur vérifie chaque `file:ligne` contre le diff :
  issue hors périmètre → affichée DÉCLASSÉE « non ancrée », jamais
  supprimée en silence ; la correction guidée l'ignore. Zéro appel en plus,
  la dégradation est visible.

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
- Réduction du verdict swarm (dogfood Q1, 3/3) : table devil-spec-swarm
  explicitée — ≥ 2 approve ET 0 reject → VALABLE ; ≥ 2 reject → JETABLE ;
  tout le reste (dont le split approve/rework/reject) → MODIFICATIONS
  REQUISES. Pas de score global consolidé : grille par devil + moyenne
  indicative. **Garde-fou sécurité** : toute issue `critical` de catégorie
  `security` portée par au moins un devil plafonne le verdict à
  MODIFICATIONS REQUISES, même à 3 approve.

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

## Questions écartées au dogfood (2026-07-18)

Interrogatoire à trois voix (`/devil-brain-swarm`) : 10 questions consolidées,
5 retenues (traitées dans les sections amendées ci-dessus), 5 écartées avec
rationale — l'esquive visible plutôt que silencieuse :

1. **Working tree mouvant pendant la review** (gemini, bloquante) : outil
   mono-utilisateur en session interactive ; Romain est dans la session
   pendant le run, et la correction guidée relit chaque fichier avant Edit.
   Pas de CI, pas d'équipe : géré par l'usage.
2. **Panne fournisseur / quorum** (deepseek + glm, importante) : déjà couvert
   par le pattern v0.2.0 décalqué — 3 voix pleines, 2 voix signalées en tête
   de rapport, ≤ 1 voix = pas de verdict. Jamais de dégradation silencieuse.
3. **Refs git indisponibles / offline** (gemini, importante) : outil local
   interactif ; une erreur git remonte à l'orchestrateur qui la présente et
   propose un target alternatif. Pas de mode headless à protéger.
4. **Temp files orphelins sur crash** (gemini, importante) : même pattern
   mktemp + trash que v0.2.0, résidu sur crash toléré sur machine perso.
5. **Version pinning des modèles** (deepseek, importante) : les modèles cloud
   (agy, ollama cloud) ne sont pas pinnables par le plugin ; dérive assumée,
   le dogfood continu fait office de non-régression. Limite v0.2.0 déjà actée
   (« diversité des trois voix postulée »).

## Points d'attention livraison

- Collision de nom réglée : la skill perso `~/.claude/skills/devil-code`
  (v0 Antigravity) a été trashée le 2026-07-18.
- MàJ plugin = uninstall + install (gotcha connu).
- Bump manifest 0.3.0 ; README et `_memory_` à mettre à jour à
  l'implémentation.
