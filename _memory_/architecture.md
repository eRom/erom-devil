# Architecture — erom-agence-devil

> MàJ : 2026-07-18

**Type** : Plugin Claude Code `devil` (v0.1.0), distribué par `erom-marketplace`.

**Objectif** : « avocats du diable » qui jugent une spec technique contre son
brainstorm d'origine (dérives, manques, incohérences) AVANT implémentation.
Le demandeur est généralement Claude lui-même → sorties strictement parseables.

**Stack** : agents + skills en markdown ; bash + `jq` + `sed` ; `agy`
(Antigravity CLI → Gemini) ; `claude` CLI → ollama cloud (GLM/Deepseek) ;
`trash`.

**Les 3 devils** :
| devil | modèle | transport |
|---|---|---|
| gemini | Gemini 3.5 Flash (High) | agy (écrit la review en fichier, bug stdout #76) |
| glm | glm-5.2:cloud | claude -p sur ollama cloud (texte pur, JSON sur stdout) |
| deepseek | deepseek-v4-pro:cloud | idem glm (fichier dérivé par sed) |

**Arborescence** :
```
.claude-plugin/plugin.json   manifest (name devil)
agents/devil-spec-{gemini,glm,deepseek}.md   wrappers Sonnet symétriques
skills/devil-spec/           unitaire (choix du devil, gemini défaut)
skills/devil-spec-swarm/     tribunal : 3 devils parallèles + synthèse
scripts/spec-review-schema.json   contrat JSON (3 verdicts)
scripts/devil-mission.md     mission commune (6 critères)
examples/                    fixtures de smoke (6 défauts plantés)
.specs/plugin-devil/         brainstorming + architecture-technique + plan
```

**Flux** : skill détecte brainstorm+specs → spawn agent(s) → chaque agent
retourne une enveloppe JobJSON `{devil, model, status, review|error}` →
skill présente le rapport (unitaire) ou consolide (swarm).

**Entrées** : toujours 2 fichiers (brainstorming + specs). Jamais un plan
(décision Romain : les devils voient les specs, or on n'y met pas de secrets).

**Déps externes critiques** : `agy` authentifié (gemini) ; ollama local avec
accès cloud (glm/deepseek) ; `jq`, `trash`.

**Prochain chantier** : « devil brain » (non spécifié à ce jour).
