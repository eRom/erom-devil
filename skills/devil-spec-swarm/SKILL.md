---
name: devil-spec-swarm
description: "Tribunal des avocats du diable : Gemini + GLM + Deepseek reviewent les specs en parallèle, synthèse consolidée avec verdict VALABLE / MODIFICATIONS REQUISES / JETABLE. Triggers: /devil-spec-swarm, 'swarm de review', 'tribunal des specs'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-spec-swarm — Le tribunal des trois devils

Les 3 avocats du diable (Gemini via agy, GLM et Deepseek via ollama cloud)
jugent les mêmes specs EN PARALLÈLE. Toi (l'orchestrateur) tu consolides :
convergences, dissonances, verdict final argumenté.

## Étape 0 — Chemins du plugin

Identique à /devil-spec : racine = deux niveaux au-dessus du base directory
injecté ; résous `SCHEMA_FILE` = `<racine>/scripts/devil-spec-schema.json`
et `MISSION_FILE` = `<racine>/scripts/devil-spec-mission.md`, vérifie l'existence.

## Étape 1 — Fichiers d'entrée

Même détection, mêmes règles et même confirmation que /devil-spec (paths en
argument sinon auto-detect `.specs/`), avec l'annonce :

> **Tribunal convoqué :** gemini + glm + deepseek en parallèle (jusqu'à 9 min).
> Fichiers : <brainstorm> vs <specs>. Je lance ?

## Étape 2 — Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message (c'est ce qui les
fait tourner en parallèle). Même prompt pour les trois, seul le
subagent_type change : `devil:devil-gemini`, `devil:devil-glm`,
`devil:devil-deepseek` (si le type est introuvable, fallback sans préfixe) :

```
prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.verdict | IN(\"approve\",\"rework\",\"reject\"))\nINPUTS:\nBRAINSTORMING:<abs brainstorm>\nSPECS:<abs specs>\n\nExécute la procédure de transport."
```

## Étape 3 — Collecte et quorum

Chaque retour est une enveloppe `{devil, model, status, review|error}`.

- 3 voix `ok` → synthèse pleine.
- 2 voix `ok` → continue, et le rapport OUVRE sur la voix absente :
  « ⚠ <devil> n'a pas rendu son verdict (<error> — <detail court>). Synthèse
  sur 2 voix. »
- ≤ 1 voix `ok` → PAS de verdict. Rapport d'échec avec le détail des erreurs,
  proposer : relancer le swarm, ou basculer en unitaire (/devil-spec <devil>).

Un retour qui n'est pas une enveloppe JSON valide compte comme voix absente
(ne JAMAIS interpréter un texte d'erreur comme une review).

## Étape 4 — Synthèse (ton travail d'orchestrateur)

### Verdict final

| Situation (sur les voix exprimées) | Verdict |
|---|---|
| ≥ 2 approve et 0 reject | **VALABLE** |
| ≥ 2 reject | **JETABLE** |
| tout le reste | **MODIFICATIONS REQUISES** |

### Consolidation des issues

- Regroupe les issues des devils par PROBLÈME DE FOND (même problème ≠ mêmes
  mots — c'est un jugement sémantique, pas un match textuel).
- Badge de convergence : 3/3, 2/3, 1/3 (sur les voix exprimées : à 2 voix,
  2/2 et 1/2).
- Tri : convergence décroissante, puis sévérité max du groupe.
- Garde la suggestion la plus actionnable du groupe ; sévérité = la plus haute.

### Voix dissonante

Si un devil s'écarte des deux autres (ex. reject isolé contre 2 approve, ou
écart de score > 30 sur un critère), section dédiée : qui, sur quoi, son
argument principal. La majorité ne l'écrase JAMAIS silencieusement.

## Étape 5 — Rapport

```
══════ TRIBUNAL DES DEVILS ══════

[⚠ voix absente le cas échéant]

Verdict : [VALABLE ✓ | MODIFICATIONS REQUISES ⚠ | JETABLE ☠]
<2-3 phrases d'argumentation : pourquoi ce verdict, à partir des verdicts
individuels et des convergences>

Scores :
  Critère       gemini  glm  deepseek  (moyenne)
  Fidélité        <n>   <n>    <n>       <n>
  Complétude      <n>   <n>    <n>       <n>
  Cohérence       <n>   <n>    <n>       <n>
  Faisabilité     <n>   <n>    <n>       <n>
  Sécurité        <n>   <n>    <n>       <n>
  Clarté          <n>   <n>    <n>       <n>
  GLOBAL          <n>   <n>    <n>       <n>

Verdicts : gemini <verdict> · glm <verdict> · deepseek <verdict>
```

Issues consolidées (tableau) :

```
| Conv | Sév | Cat | Problème | Suggestion | Devils |
| 3/3  | critical | fidelity | … | … | gemini, glm, deepseek |
```

Puis la section « Voix dissonante » si applicable.

## Étape 6 — Next steps selon le verdict

### VALABLE
> Le tribunal valide. [Mentionner les issues 2/3+ restantes s'il y en a]
> → On continue le flow (plan d'implémentation) ?

### MODIFICATIONS REQUISES
> 1. **Corriger** — j'adresse les issues consolidées, convergentes (3/3, 2/3)
>    d'abord, puis critical/high isolées
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-swarm** — je corrige puis je reconvoque le tribunal (1 max)

### JETABLE
> Le tribunal enterre la spec : <les 1-2 raisons de fond convergentes>.
> Pas de correction incrémentale : on rouvre le brainstorm, puis specs neuves.

## Règles

- Ne JAMAIS modifier le brainstorm.
- Corrections : mêmes règles que /devil-spec (Edit specs uniquement, low
  ignorées, pas d'invention).
- Maximum 1 re-swarm ; ensuite Romain tranche.
- Coût : 3 modèles en parallèle ≈ la durée du plus lent. C'est un gate de
  qualité, pas un lint : ne pas le déclencher sur un brouillon.
