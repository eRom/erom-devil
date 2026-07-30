---
name: brain
description: "Interrogatoire socratique d'un doc de brainstorming par un avocat du diable au choix (Gemini par défaut, GLM, Deepseek, Opus, Kimi) : les 5 questions les plus dangereuses jamais posées, sans score ni verdict. Triggers: /erom-devil:brain, 'questionne le brainstorm', 'angles morts du brainstorming'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /erom-devil:brain — L'interrogatoire socratique du brainstorming

Un devil externe lit un doc de brainstorming (définition de besoins) et rend
les questions les PLUS DANGEREUSES jamais posées — 5 max, sans score ni
verdict. Toi (l'orchestrateur) tu restitues, Romain écarte ou retient, et tu
amendes le doc au fil de ses réponses.

## Syntaxe

```
/erom-devil:brain                          # auto-detect, devil gemini
/erom-devil:brain glm                      # auto-detect, devil glm
/erom-devil:brain chemin/brainstorming.md  # fichier explicite, gemini
/erom-devil:brain chemin/brainstorming.md deepseek
```

## Étape 0 — Résoudre le devil et les chemins du plugin

- Devil : l'argument s'il vaut `gemini`, `glm`, `deepseek`, `opus` ou `kimi` ;
  sinon `gemini`.
- Racine du plugin : deux niveaux au-dessus du « Base directory for this
  skill » injecté ci-dessus. Résous en absolu :
  - `SCHEMA_FILE` = `<racine>/scripts/devil-brain-schema.json`
  - `MISSION_FILE` = `<racine>/scripts/devil-brain-mission.md`
- Vérifie que les deux existent (Read). S'ils manquent, arrête et signale un
  plugin corrompu.

## Étape 1 — Identifier le doc de brainstorming

- Path fourni en argument → vérifie qu'il existe et n'est pas vide.
  Introuvable ou vide → arrête et demande, AUCUN appel modèle.
- Sinon auto-detect : Glob `**/brainstorming.md` dans `.specs/` ; plusieurs
  chantiers → demande lequel ; un seul → prends-le.
- En pleine session de brainstorming sans fichier : écris d'abord l'état
  courant de la compréhension dans `.specs/<chantier>/brainstorming.md`,
  puis continue avec ce fichier.

Confirmation avant lancement :

> **Interrogatoire :** <devil> questionne `<fichier>` (jusqu'à 9 min). Go ?

## Étape 2 — Lancer le sous-agent

Spawn `erom-devil:<devil>` ; si ce type est introuvable (plugin non
chargé), retente `<devil>` sans préfixe :

```
Agent(
  subagent_type: "erom-devil:<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"assessment\") and has(\"questions\") and (.questions|type==\"array\" and length<=5) and (.questions|all(has(\"question\") and has(\"domain\") and has(\"risk\") and (.criticality|IN(\"blocking\",\"important\",\"exploratory\"))))\nINPUTS:\nBRAINSTORMING:<abs>\n\nExécute la procédure de transport."
)
```

Annonce avant le spawn : « **Interrogatoire en cours…** <devil> cherche les
questions jamais posées (jusqu'à 9 min). »

## Étape 3 — Parser l'enveloppe et restituer

Le retour de l'agent est UNE ligne JSON : `{devil, model, status,
review|error+detail}`.

### Si `status: "error"`
```
══════ INTERROGATOIRE ÉCHOUÉ ══════

Devil : <devil> (<model>)
Erreur : <error> — <detail>

→ Relance (/erom-devil:brain <devil>), autre devil (/erom-devil:brain glm|deepseek|gemini),
  ou les trois voix (/erom-devil:brain-swarm).
```

### Si `status: "ok"` et `questions` vide
> **Rien de dangereux à signaler.** <assessment>
> Le devil n'a pas de question à poser : tu peux passer aux specs.

(Pas de Q&A forcé. Fin de la skill.)

### Si `status: "ok"` avec questions
```
══════ DEVIL BRAIN ══════

Devil : <devil> (<model>)
Assessment : <assessment>

| # | Criticité | Domaine | Question | Risque si on spécifie sans répondre |
```
Tri : blocking > important > exploratory. Criticités affichées en français :
BLOQUANTE / IMPORTANTE / EXPLORATOIRE.

## Étape 4 — Tri puis Q&A ciblé

1. Demande à Romain lesquelles il ÉCARTE (réponse d'un mot : « 2 et 5 »,
   « aucune », « toutes »…).
2. Questions écartées : propose de les tracer en non-buts explicites du doc
   de brainstorming. Romain tranche ; si oui, ajoute chaque sujet écarté à
   la section Non-buts via Edit.
3. Questions retenues : pose-les UNE PAR UNE. Après chaque réponse de
   Romain, amende le doc de brainstorming via Edit — la décision s'intègre
   dans la section qu'elle concerne (ou une nouvelle section), jamais en
   annexe fourre-tout.
4. Fin : récap une ligne par question — intégrée (où) ou écartée (non-but ou
   simple abandon).

## Règles

- Tu ne modifies QUE le doc de brainstorming, et uniquement avec les
  réponses de Romain — jamais tes propres réponses aux questions.
- Pas de re-passage automatique après amendement : le re-appel est toujours
  manuel (/erom-devil:brain ou /erom-devil:brain-swarm).
- Le devil ne modifie rien : c'est toi qui amendes.
