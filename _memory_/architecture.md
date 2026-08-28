# Architecture — erom-devil (dossier local : erom-agence-devil)

> MàJ : 2026-08-20 (v0.7.0)

**Type** : Plugin Claude Code `erom-devil` (renommé le 2026-07-30, ex-`devil`),
distribué par `erom-marketplace`.

**Objectif** : « avocats du diable » externes sur le travail, du document
amont à la porte de merge. Quatre exercices :
- **spec** : juger une spec technique contre son brainstorm (score, verdict
  approve/rework/reject, issues) — unitaire ou swarm (VALABLE/MODIFS/JETABLE).
- **brain** : interrogatoire socratique d'un brainstorming seul — les 5
  questions les plus dangereuses jamais posées, SANS score ni verdict ;
  0 question = prêt à spécifier (signal faible, limite actée).
- **code** : review d'un CHANGEMENT (PR/branche/range/working tree) packagé
  hermétiquement (DIFF + FILES + INTENT opt.), scan anti-fuite pré-vol,
  ancrage file:ligne vérifié au retour, garde-fou sécurité en swarm (opus et
  kimi exclus du tribunal, dispo en unitaire).
- **review** : porte de merge complète (4e exercice, v0.7.0) : porte
  déterministe avant tout appel modèle, chasse à l'intention, input STACK
  (grille `scripts/stacks/nextjs.md`), passe de vérification orchestrateur
  (Confirmée/Réfutée/Hypothèse + balayage frontières), verdict GO/NO-GO,
  rapport persistant `docs/reviews/`. Skills review{,-swarm} par référence
  aux étapes de code{,-swarm} ; mission/schéma propres ; agents inchangés.
  Depuis le 2026-08-20, l'exercice porte une asymétrie assumée entre ses
  deux acteurs : le devil élargit son rappel sur l'axe sécurité (mission),
  l'orchestrateur élargit son filet en conséquence (Étape 9 : grille de
  réfutation à 8 motifs, périmètre couvrant TOUTE issue `security`
  ancrée). Les deux moitiés ne se modifient pas séparément. Le schéma est
  inchangé : c'est un changement de contrat textuel, pas de structure.

**Stack** : agents + skills en markdown ; bash + `jq` + `sed` ; `agy`
(Antigravity CLI → Gemini) ; `claude -p` → ollama cloud (GLM/Deepseek) ;
`trash`.

**Les 5 agents = transport PUR** (ne connaissent pas l'exercice) :
| agent | modèle | transport |
|---|---|---|
| gemini | Gemini 3.7 Flash (High) | agy (review par fichier, bug stdout #76) |
| glm | glm-5.3-flash:cloud[1m] | claude -p ollama cloud (JSON stdout) |
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

**Arborescence** (depuis v0.5.1 : le plugin livrable vit sous `plugin/`, la
marketplace le tire en `source: git-subdir` + `path: plugin` pour que `.specs/`
et `_memory_/` ne soient pas clonés dans le harnais) :
```
plugin/.claude-plugin/plugin.json  manifest (PAS de clé agents)
plugin/agents/{gemini,glm,deepseek,opus,kimi}.md transport pur (opus+kimi hors swarms)
plugin/skills/spec{,-swarm}/    exercice spec (2 inputs BRAINSTORMING+SPECS)
plugin/skills/brain{,-swarm}/   exercice brain (1 input BRAINSTORMING)
plugin/skills/code{,-swarm}/    exercice code (DIFF + FILES/INTENT opt.)
plugin/skills/review{,-swarm}/  porte de merge (DIFF + FILES/INTENT/STACK opt.)
plugin/scripts/devil-{spec,brain,code,review}-{mission.md,schema.json}
plugin/scripts/stacks/nextjs.md grille de stack Next 16 / Prisma 7 (input STACK)
plugin/README.md
examples/                       fixtures veilleur (6 défauts), code (5 défauts
                                + secret), review-stack (3 violations de grille)
.specs/plugin-devil{,-brain,-code,-review}/  designs v0.1.0, v0.2.0, v0.3.0,
                                v0.7.0 (hors plugin)
_memory_/                       cette mémoire (hors plugin)
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
