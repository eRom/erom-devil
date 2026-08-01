---
name: code-swarm
description: "Tribunal du code : Gemini + GLM + Deepseek reviewent le même changement en parallèle, consolidation par problème de fond, verdict VALABLE / MODIFICATIONS REQUISES / JETABLE avec garde-fou sécurité. Triggers: /erom-devil:code-swarm, 'swarm sur le code', 'tribunal du code'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Bash, Agent, AskUserQuestion, Edit
---

# /erom-devil:code-swarm — Le tribunal du code

Les 3 devils (gemini + glm + deepseek — ni opus ni kimi ne sont convoqués
au tribunal, choix acté) jugent le MÊME changement EN PARALLÈLE. Toi
(l'orchestrateur) tu consolides : ancrage par voix, convergences, verdict
avec garde-fou sécurité.

## Étapes 0 à 4 — identiques à /erom-devil:code

Chemins plugin, résolution du target (mêmes formes, gardes et cas
limites), packaging (mêmes règles DIFF/FILES/INTENT, un SEUL TMP_DIR
partagé en lecture par les 3 spawns), scan pré-vol (obligatoire, jamais
sauté) : déroule les Étapes 0 à 4 de `skills/code/SKILL.md`, à une
différence près — l'annonce :

> **Tribunal du code :** gemini + glm + deepseek en parallèle (jusqu'à
> 9 min). Cible : <mode/cible>. Je lance ?

## Étape 5 — Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message. Prompt
IDENTIQUE pour les trois, seul le subagent_type change :
`erom-devil:gemini`, `erom-devil:glm`, `erom-devil:deepseek` (fallback
sans préfixe si un type est introuvable) :

```
Agent(
  subagent_type: "erom-devil:<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.score|type==\"number\" and .>=0 and .<=100) and (.verdict|IN(\"approve\",\"rework\",\"reject\")) and (.criteria|has(\"correctness\") and has(\"architecture\") and has(\"security\") and has(\"performance\") and has(\"tests\") and has(\"maintainability\")) and ([.criteria.correctness,.criteria.architecture,.criteria.security,.criteria.performance,.criteria.tests,.criteria.maintainability]|all(type==\"object\" and has(\"score\") and has(\"comment\") and (.score|type==\"number\" and .>=0 and .<=100))) and (.issues|type==\"array\" and all(has(\"severity\") and has(\"category\") and has(\"file\") and has(\"description\") and has(\"failure_scenario\") and has(\"suggestion\") and (.severity|IN(\"critical\",\"high\",\"medium\",\"low\")) and (.category|IN(\"correctness\",\"architecture\",\"security\",\"performance\",\"tests\",\"maintainability\",\"intent\"))))\nINPUTS:\nDIFF:<abs diff>\nFILES:<abs files>\nINTENT:<abs intent>\n\nExécute la procédure de transport."
)
```

(`FILES:` et `INTENT:` seulement si les fichiers existent, comme en
unitaire.)

## Étape 6 — Collecte et quorum

- 3 voix `ok` → synthèse pleine.
- 2 voix `ok` → continue, le rapport OUVRE sur la voix absente :
  « ⚠ <devil> muet (<error> — <detail court>). Synthèse sur 2 voix. »
- ≤ 1 voix `ok` → PAS de verdict. Rapport d'échec, proposer : relancer le
  swarm, ou l'unitaire (/erom-devil:code <target> <devil>).

Un retour qui n'est pas une enveloppe JSON valide compte comme voix
absente (ne JAMAIS interpréter un texte d'erreur comme une review).

## Étape 7 — Ancrage par voix, puis consolidation

D'abord l'ancrage de /erom-devil:code Étape 6, appliqué PAR VOIX (les
DÉCLASSÉES d'une voix sortent de sa contribution avant consolidation, et
restent listées en « Non ancrées » avec leur devil).

Puis consolidation par PROBLÈME DE FOND — ton travail LLM natif, AUCUN
algorithme, embedding ni seuil à coder. Heuristiques d'aide au jugement :
- candidats à l'équivalence : même fichier ET plages de lignes
  chevauchantes ou voisines (±10) ET même problème de fond (la catégorie
  aide mais ne décide pas — un même bug peut être classé correctness par
  l'un, security par l'autre) ;
- deux issues au même endroit peuvent rester deux problèmes distincts ;
  en cas de doute, NE PAS fusionner ;
- badge de convergence 3/3, 2/3, 1/3 (sur les voix exprimées : à 2 voix,
  2/2 et 1/2) ; sévérité du groupe = la plus haute ; suggestion la plus
  actionnable conservée.
- Tri : convergence décroissante, puis sévérité max du groupe.

## Étape 8 — Verdict

| Situation (voix exprimées) | Verdict |
|---|---|
| ≥ 2 approve ET 0 reject | **VALABLE** |
| ≥ 2 reject | **JETABLE** |
| tout le reste (dont split approve/rework/reject) | **MODIFICATIONS REQUISES** |

**Garde-fou sécurité** : ≥ 1 issue `critical` de catégorie `security`
ANCRÉE portée par au moins un devil → le verdict ne peut pas être
VALABLE : un VALABLE issu de la table est ramené à MODIFICATIONS
REQUISES avec mention explicite du garde-fou ; un JETABLE reste JETABLE
(le garde-fou ne remonte jamais un verdict).

Pas de score consolidé : grille des scores par devil + moyenne indicative.

## Étape 9 — Rapport

```
══════ TRIBUNAL DU CODE ══════

[⚠ voix absente le cas échéant]
[REVIEW SUR DIFF SEUL — contexte réduit]        ← si FILES omis

Verdict : [VALABLE ✓ | MODIFICATIONS REQUISES ⚠ | JETABLE ☠]
[— garde-fou sécurité appliqué, le cas échéant]
<2-3 phrases d'argumentation à partir des verdicts et convergences>

Scores :
  Critère          gemini  glm  deepseek  (moyenne)
  Correctness        <n>   <n>    <n>       <n>
  Architecture       <n>   <n>    <n>       <n>
  Security           <n>   <n>    <n>       <n>
  Performance        <n>   <n>    <n>       <n>
  Tests              <n>   <n>    <n>       <n>
  Maintainability    <n>   <n>    <n>       <n>
  GLOBAL             <n>   <n>    <n>       <n>

Verdicts : gemini <verdict> · glm <verdict> · deepseek <verdict>
```

Issues consolidées :

```
| Conv | Sév | Cat | Fichier | Problème | Suggestion | Devils |
```

Puis « Non ancrées » (avec devil d'origine) et « Voix dissonante » si
applicable (reject isolé, écart de score > 30 sur un critère : qui, sur
quoi, son argument principal — la majorité ne l'écrase JAMAIS
silencieusement).

## Étape 10 — Next steps selon le verdict

### VALABLE
> Le tribunal valide. [Mentionner les issues 2/3+ restantes]
> → Prêt à commit/merge ?

### MODIFICATIONS REQUISES
> 1. **Corriger** — correction guidée (si le mode l'autorise) : issues
>    convergentes (3/3, 2/3) d'abord, puis critical/high isolées
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-swarm** — je corrige puis je reconvoque le tribunal (1 max)

### JETABLE
> Le tribunal enterre le changement : <les 1-2 raisons de fond
> convergentes>. Pas de correction incrémentale : on rouvre la conception.

## Règles

- Correction guidée : mêmes règles et même séquence que /erom-devil:code
  Étape 8 (modes autorisés, low ignorées, non-ancrées exclues, le devil
  ne modifie jamais rien).
- Maximum 1 re-swarm ; ensuite Romain tranche.
- opus et kimi ne siègent pas au tribunal : ce sont des juges indépendants,
  appelés unitairement (/erom-devil:code opus, /erom-devil:code kimi).
- `trash "$TMP_DIR"` en fin de run.
- Coût : 3 modèles en parallèle ≈ la durée du plus lent. C'est un gate de
  commit/merge, pas un lint : pas sur un diff de deux lignes.
