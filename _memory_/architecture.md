# Architecture — erom-agence-devil

> MàJ : 2026-07-18 (v0.3.0)

**Type** : Plugin Claude Code `devil` (v0.3.0), distribué par `erom-marketplace`.

**Objectif** : « avocats du diable » externes sur les documents amont, AVANT
implémentation. Deux exercices :
- **spec** : juger une spec technique contre son brainstorm (score, verdict
  approve/rework/reject, issues) — unitaire ou swarm (VALABLE/MODIFS/JETABLE).
- **brain** : interrogatoire socratique d'un brainstorming seul — les 5
  questions les plus dangereuses jamais posées, SANS score ni verdict ;
  0 question = prêt à spécifier (signal faible, limite actée).
- **code** : review d'un CHANGEMENT (PR/branche/range/working tree) packagé
  hermétiquement (DIFF + FILES + INTENT opt.), scan anti-fuite pré-vol,
  ancrage file:ligne vérifié au retour, garde-fou sécurité en swarm (opus
  exclu du tribunal, dispo en unitaire).

**Stack** : agents + skills en markdown ; bash + `jq` + `sed` ; `agy`
(Antigravity CLI → Gemini) ; `claude -p` → ollama cloud (GLM/Deepseek) ;
`trash`.

**Les 3 agents = transport PUR** (ne connaissent pas l'exercice) :
| agent | modèle | transport |
|---|---|---|
| devil-gemini | Gemini 3.5 Flash (High) | agy (review par fichier, bug stdout #76) |
| devil-glm | glm-5.2:cloud | claude -p ollama cloud (JSON stdout) |
| devil-deepseek | deepseek-v4-pro:cloud | idem glm (jumeau sed) |

L'exercice est porté par les SKILLS via le contrat de spawn :
MISSION_FILE + SCHEMA_FILE + VALIDATE_JQ + INPUTS étiquetés (`LABEL:abs`).
Ajouter un exercice = 1 mission + 1 schéma + 2 skills, agents inchangés.

**Arborescence** :
```
.claude-plugin/plugin.json      manifest 0.3.0 (PAS de clé agents)
agents/devil-{gemini,glm,deepseek,opus}.md transport pur (opus hors swarms)
skills/devil-spec{,-swarm}/     exercice spec (2 inputs BRAINSTORMING+SPECS)
skills/devil-brain{,-swarm}/    exercice brain (1 input BRAINSTORMING)
skills/devil-code{,-swarm}/     exercice code (DIFF + FILES/INTENT opt.)
scripts/devil-{spec,brain,code}-{mission.md,schema.json}
examples/                       fixtures veilleur (6 défauts) + code (5 défauts + secret)
.specs/plugin-devil{,-brain,-code}/  designs v0.1.0, v0.2.0, v0.3.0
```

**Flux** : skill résout mission/schéma/inputs → spawn agent(s)
`devil:devil-<nom>` → enveloppe `{devil, model, status, review|error}` →
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
