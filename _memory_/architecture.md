# Architecture — erom-devil (dossier local : erom-agence-devil)

> MàJ : 2026-07-18 (v0.3.0)

**Type** : Plugin Claude Code `erom-devil` (renommé le 2026-07-30, ex-`devil`),
distribué par `erom-marketplace`.

**Objectif** : « avocats du diable » externes sur les documents amont, AVANT
implémentation. Trois exercices :
- **spec** : juger une spec technique contre son brainstorm (score, verdict
  approve/rework/reject, issues) — unitaire ou swarm (VALABLE/MODIFS/JETABLE).
- **brain** : interrogatoire socratique d'un brainstorming seul — les 5
  questions les plus dangereuses jamais posées, SANS score ni verdict ;
  0 question = prêt à spécifier (signal faible, limite actée).
- **code** : review d'un CHANGEMENT (PR/branche/range/working tree) packagé
  hermétiquement (DIFF + FILES + INTENT opt.), scan anti-fuite pré-vol,
  ancrage file:ligne vérifié au retour, garde-fou sécurité en swarm (opus et
  kimi exclus du tribunal, dispo en unitaire).

**Stack** : agents + skills en markdown ; bash + `jq` + `sed` ; `agy`
(Antigravity CLI → Gemini) ; `claude -p` → ollama cloud (GLM/Deepseek) ;
`trash`.

**Les 5 agents = transport PUR** (ne connaissent pas l'exercice) :
| agent | modèle | transport |
|---|---|---|
| gemini | Gemini 3.5 Flash (High) | agy (review par fichier, bug stdout #76) |
| glm | glm-5.2:cloud[1m] | claude -p ollama cloud (JSON stdout) |
| deepseek | deepseek-v4-pro:cloud[1m] | idem glm (jumeau sed) |
| opus | Opus 4.8 xHigh | claude -p (hors swarms, unitaire seulement) |
| kimi | kimi-k3:cloud[1m] | idem glm (jumeau sed) ; hors swarms, unitaire |

Le suffixe `[1m]` (contexte 1M) est porté par les 3 transports ollama, dans
la ligne d'appel ET dans le champ `model` de l'enveloppe. Il DOIT être quoté
dans le bash : le Bash tool tourne sous zsh, où `[1m]` non quoté est un glob
`nomatch` qui avorte la commande avant tout appel (voir gotchas.md).

L'exercice est porté par les SKILLS via le contrat de spawn :
MISSION_FILE + SCHEMA_FILE + VALIDATE_JQ + INPUTS étiquetés (`LABEL:abs`).
Ajouter un exercice = 1 mission + 1 schéma + 2 skills, agents inchangés.

**Arborescence** :
```
.claude-plugin/plugin.json      manifest 0.3.0 (PAS de clé agents)
agents/{gemini,glm,deepseek,opus,kimi}.md transport pur (opus+kimi hors swarms)
skills/spec{,-swarm}/           exercice spec (2 inputs BRAINSTORMING+SPECS)
skills/brain{,-swarm}/          exercice brain (1 input BRAINSTORMING)
skills/code{,-swarm}/           exercice code (DIFF + FILES/INTENT opt.)
scripts/devil-{spec,brain,code}-{mission.md,schema.json}
examples/                       fixtures veilleur (6 défauts) + code (5 défauts + secret)
.specs/plugin-devil{,-brain,-code}/  designs v0.1.0, v0.2.0, v0.3.0
```

**Flux** : skill résout mission/schéma/inputs → spawn agent(s)
`erom-devil:<nom>` → enveloppe `{devil, model, status, review|error}` →
skill restitue (spec : rapport scoré ; brain : tableau questions puis tri +
Q&A qui amende le doc de brainstorming).

**Limites actées (dogfood 2026-07-18, section « Limites connues » du
brainstorming v0.2.0)** : 0-question non calibré, diversité des 3 modèles
postulée, tri mono-personne, tension secret↔richesse, re-passage stateless.

**Pistes v0.3** : parsing de docs non textuels (Mermaid), contrat du draft
figé en session. + Minor différé : clé d'enveloppe `review` porte aussi la
sortie brain (renommage `output` un jour, cosmétique).

**Déps externes critiques** : `agy` authentifié ; ollama local avec accès
cloud ; `jq`, `trash`.
