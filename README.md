# devil — avocats du diable pour la définition de besoins

Plugin Claude Code. Trois reviewers critiques externes (Gemini, GLM,
Deepseek) attaquent tes documents amont, AVANT implémentation, sous deux
angles :

- **devil-spec** : ils jugent une spec technique contre son brainstorm
  d'origine (dérives, manques, incohérences), score et verdict à la clé.
- **devil-brain** : ils interrogent un brainstorming seul et rendent les
  questions les plus dangereuses jamais posées (angles morts, pans oubliés),
  sans score ni verdict.

## Les devils

| Devil | Modèle | Transport |
|---|---|---|
| gemini | Gemini 3.5 Flash (High) | Antigravity CLI (agy) |
| glm | glm-5.2:cloud | claude CLI → ollama cloud |
| deepseek | deepseek-v4-pro:cloud | claude CLI → ollama cloud |

## Usage

```
/devil-spec                          # specs vs brainstorm, devil gemini
/devil-spec brainstorm.md specs.md glm
/devil-spec-swarm                    # les 3 devils en parallèle + synthèse
/devil-brain                         # questions socratiques sur un brainstorming
/devil-brain brainstorming.md deepseek
/devil-brain-swarm                   # les 3 voix, consolidation par convergence
```

**devil-spec** — entrées : 2 fichiers (brainstorming + specs). Sortie par
devil : JSON strict (score 0-100, verdict approve/rework/reject, 6 critères,
issues actionnables). Le swarm consolide : VALABLE, MODIFICATIONS REQUISES,
ou JETABLE, avec convergence des issues (3/3, 2/3, 1/3) et voix dissonantes.

**devil-brain** — entrée : 1 fichier (brainstorming). Sortie par devil : les
5 questions les plus dangereuses jamais posées (domaine, risque, criticité)
plus une impression en une ligne — sans score ni verdict, 0 question = prêt
à spécifier. Le swarm consolide par convergence, tri criticité puis
convergence, puis Q&A ciblé qui amende le doc.

## Prérequis

- `agy` (Antigravity CLI) authentifié, pour le devil gemini.
- `claude` CLI + ollama local avec accès aux modèles cloud (`glm-5.2:cloud`,
  `deepseek-v4-pro:cloud`), pour glm et deepseek.
- `jq`, `trash`.

## Installation

Via la marketplace eRom : `claude plugin install devil@erom-marketplace`.
