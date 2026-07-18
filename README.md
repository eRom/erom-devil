# devil — avocats du diable pour specs techniques

Plugin Claude Code. Trois reviewers critiques externes jugent une spec
technique contre son brainstorm d'origine : dérives, manques, incohérences,
AVANT implémentation.

## Les devils

| Devil | Modèle | Transport |
|---|---|---|
| gemini | Gemini 3.5 Flash (High) | Antigravity CLI (agy) |
| glm | glm-5.2:cloud | claude CLI → ollama cloud |
| deepseek | deepseek-v4-pro:cloud | claude CLI → ollama cloud |

## Usage

```
/devil-spec                          # auto-detect dans .specs/, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
```

Entrées : toujours 2 fichiers (brainstorming + specs). Sortie par devil :
JSON strict (score 0-100, verdict approve/rework/reject, 6 critères, issues
actionnables). Le swarm consolide : VALABLE, MODIFICATIONS REQUISES, ou
JETABLE, avec convergence des issues (3/3, 2/3, 1/3) et voix dissonantes.

## Prérequis

- `agy` (Antigravity CLI) authentifié, pour le devil gemini.
- `claude` CLI + ollama local avec accès aux modèles cloud (`glm-5.2:cloud`,
  `deepseek-v4-pro:cloud`), pour glm et deepseek.
- `jq`, `trash`.

## Installation

Via la marketplace eRom : `claude plugin install devil@erom-marketplace`.
