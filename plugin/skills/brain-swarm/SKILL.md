---
name: brain-swarm
description: "Interrogatoire socratique à trois voix : Gemini + GLM + Deepseek questionnent le même doc de brainstorming en parallèle, questions consolidées par convergence (3/3, 2/3, 1/3), sans score ni verdict. Triggers: /erom-devil:brain-swarm, 'swarm socratique', 'les devils sur le brainstorm'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /erom-devil:brain-swarm — L'interrogatoire à trois voix

Les 3 devils lisent le MÊME doc de brainstorming EN PARALLÈLE et rendent
chacun leurs 5 questions les plus dangereuses. Toi (l'orchestrateur) tu
consolides — équivalences, convergence, tri — puis tri + Q&A avec Romain.

## Étape 0 — Chemins du plugin

Identique à /erom-devil:brain : racine = deux niveaux au-dessus du base directory
injecté ; résous `SCHEMA_FILE` = `<racine>/scripts/devil-brain-schema.json`
et `MISSION_FILE` = `<racine>/scripts/devil-brain-mission.md`, vérifie
l'existence.

## Étape 1 — Doc de brainstorming

Identique à /erom-devil:brain (path explicite sinon auto-detect
`**/brainstorming.md` dans `.specs/` ; introuvable ou vide → stop sans
appel ; en pleine session sans fichier → écris d'abord le draft).
Confirmation :

> **Interrogatoire à trois voix :** gemini + glm + deepseek questionnent
> `<fichier>` en parallèle (jusqu'à 9 min). Go ?

## Étape 2 — Spawner les 3 devils EN PARALLÈLE

IMPORTANT : les 3 appels Agent partent dans UN SEUL message (c'est ce qui
les fait tourner en parallèle). Prompt IDENTIQUE pour les trois — le même
qu'à /erom-devil:brain étape 2, VALIDATE_JQ compris ; seul le subagent_type
change : `erom-devil:gemini`, `erom-devil:glm`, `erom-devil:deepseek`
(si un type est introuvable, fallback sans préfixe) :

```
Agent(
  subagent_type: "erom-devil:<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"assessment\") and has(\"questions\") and (.questions|type==\"array\" and length<=5) and (.questions|all(has(\"question\") and has(\"domain\") and has(\"risk\") and (.criticality|IN(\"blocking\",\"important\",\"exploratory\"))))\nINPUTS:\nBRAINSTORMING:<abs>\n\nExécute la procédure de transport."
)
```

## Étape 3 — Collecte et quorum

Chaque retour est une enveloppe `{devil, model, status, review|error}`.

- 3 voix `ok` → consolidation pleine.
- 2 voix `ok` → continue, et le rapport OUVRE sur la voix absente :
  « ⚠ <devil> muet (<error> — <detail court>). Consolidation sur 2 voix. »
- ≤ 1 voix `ok` → PAS de consolidation. Rapport d'échec avec le détail des
  erreurs, proposer : relancer le swarm, ou l'unitaire (/erom-devil:brain <devil>).

Un retour qui n'est pas une enveloppe JSON valide compte comme voix absente
(ne JAMAIS interpréter un texte d'erreur comme des questions).

## Étape 4 — Consolidation (ton travail d'orchestrateur)

Opérée par TOI, capacité LLM native — aucun algorithme, embedding ni seuil :

- Deux questions sont ÉQUIVALENTES si elles visent le même angle mort ou
  nomment le même risque, même formulées différemment.
- Chaque groupe : badge de convergence 3/3, 2/3, 1/3 (sur les voix
  exprimées : à 2 voix, 2/2 et 1/2) ; criticité du groupe = la plus haute
  parmi les voix groupées ; garde la formulation la plus incisive et la
  plus spécifique au projet.
- Les singletons restent au tableau avec leur badge 1/3.
- TRI : criticité d'abord (blocking > important > exploratory), convergence
  ensuite. Une bloquante 1/3 passe devant une exploratoire 3/3.

## Étape 5 — Rapport

```
══════ INTERROGATOIRE À TROIS VOIX ══════

[⚠ voix absente le cas échéant]

Assessments : gemini « … » · glm « … » · deepseek « … »

| # | Criticité | Conv | Domaine | Question | Risque | Devils |
```
Criticités affichées en français : BLOQUANTE / IMPORTANTE / EXPLORATOIRE.

Si TOUTES les voix exprimées rendent 0 question :
> **Rien à signaler : aucune voix n'a de question dangereuse.**
> Assessments : <une ligne par devil>. Tu peux passer aux specs.

(Pas de Q&A forcé. Fin de la skill.)

## Étape 6 — Tri puis Q&A ciblé

Identique à /erom-devil:brain étape 4 : Romain écarte d'un regard ; écartées →
non-buts proposés dans le doc ; retenues posées UNE PAR UNE ; doc de
brainstorming amendé via Edit au fil des réponses ; récap final une ligne
par question.

## Règles

- Tu ne modifies QUE le doc de brainstorming, et uniquement avec les
  réponses de Romain.
- Pas de re-passage automatique après amendement : re-appel manuel.
- Coût : 3 modèles en parallèle ≈ la durée du plus lent. C'est un gate de
  définition de besoins, pas un lint : pas sur un brouillon de trois lignes.
