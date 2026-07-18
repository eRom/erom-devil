# Architecture technique — Plugin « devil » v0.1.0

> Specs du chantier. Amont : `brainstorming.md` (même dossier).
> Aval : plan d'implémentation via superpowers:writing-plans.

## 1. Identité et arborescence

Plugin Claude Code `devil`, distribué par `erom-marketplace`, repo source
`github.com/eRom/erom-agence-devil`.

```
erom-agence-devil/
├── .claude-plugin/plugin.json        # name "devil", v0.1.0
├── agents/
│   ├── devil-spec-gemini.md          # devil-spec-reviewer renommé (agy)
│   ├── devil-spec-glm.md             # claude -p → ollama glm-5.2:cloud
│   └── devil-spec-deepseek.md        # claude -p → ollama deepseek-v4-pro:cloud
├── skills/
│   ├── devil-spec/SKILL.md           # unitaire, arg devil (défaut gemini)
│   └── devil-spec-swarm/SKILL.md     # les 3 en parallèle + synthèse
├── scripts/
│   ├── spec-review-schema.json       # contrat v2, partagé
│   └── devil-mission.md              # mission commune (6 critères)
├── examples/
│   ├── brainstorming.md              # fixtures smoke test, noms neutres
│   └── specs.md
└── README.md
```

Résolution des chemins : le harness injecte « Base directory for this skill »
à l'invocation d'une skill ; racine plugin = `base_dir/../..`. Les skills
passent aux agents des chemins ABSOLUS (fichiers d'entrée, SCHEMA_FILE,
MISSION_FILE). Aucun chemin `~/.claude/` codé en dur nulle part.

`plugin.json` minimal : `{ "name": "devil", "description": …, "version":
"0.1.0" }` ; champs exacts alignés au build sur un plugin.json réel du cache
local (`~/.claude/plugins/cache/`).

## 2. Contrat commun v2

### 2.1 Schéma de review (`scripts/spec-review-schema.json`)

Delta vs v1 : `verdict` enum `["approve", "rework", "reject"]`. Règle :
approve si score ≥ 80, rework si 50-79, reject si < 50. Le reste est inchangé
(6 critères scorés + commentés, issues severity/category/description/
suggestion/source). Un seul schéma pour les 3 devils.

### 2.2 Enveloppe de retour d'agent

Chaque agent termine par UN objet JSON, succès ou échec :

```json
{ "devil": "glm", "model": "glm-5.2:cloud", "status": "ok", "review": { … } }
{ "devil": "glm", "model": "glm-5.2:cloud", "status": "error",
  "error": "CLI_FAILED | PARSE_ERROR | TIMEOUT", "detail": "extrait ≤ 500 chars" }
```

L'échec est structuré et reconnaissable : le swarm ne peut jamais confondre
une erreur avec une review (leçon playbook sur les retours de sous-agents).

### 2.3 Mission commune (`scripts/devil-mission.md`)

Le texte de mission actuellement inline dans l'agent agy (architecte senior,
6 critères, règles d'exigence, consigne JSON strict) extrait en template
partagé. Placeholders remplis par chaque agent selon son mode : chemins de
fichiers pour Gemini (agy lit lui-même), contenus embarqués pour GLM/Deepseek.
Un seul endroit à éditer pour durcir les 3 devils d'un coup.

## 3. Agents

Wrappers Sonnet, `color: red`, `tools: Bash, Read, Glob, Grep` (comme
l'existant). Entrée par prompt : `BRAINSTORM_FILE`, `SPECS_FILE`,
`SCHEMA_FILE`, `MISSION_FILE`. Sortie : enveloppe 2.2, rien d'autre.

### 3.1 devil-spec-gemini (renommage de devil-spec-reviewer)

Mécanique agy conservée telle quelle : `--add-dir`, `--print` dernier flag,
`< /dev/null`, review écrite en fichier tmp par agy, timeout Bash 540000 ms,
1 retry, nettoyage trash. Deltas : schéma v2 (3 états), mission lue depuis
MISSION_FILE, schéma depuis SCHEMA_FILE, enveloppe de retour 2.2.

### 3.2 devil-spec-glm / devil-spec-deepseek

Différence unique entre les deux fichiers : le nom du modèle. Procédure :

1. `realpath` des 2 fichiers d'entrée, lecture de leurs CONTENUS.
2. Assemblage du prompt dans un fichier tmp : mission + schéma inline +
   contenus délimités par balises BEGIN/END explicites.
3. Appel, au plus proche de la ligne validée par Romain :

```bash
ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434 \
ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max \
claude --model glm-5.2:cloud --dangerously-skip-permissions \
  -p --output-format json --strict-mcp-config [hermétisme] < "$PROMPT_FILE"
```

   Ajouts vs ligne validée, les SEULS points à valider au build :
   - `--output-format json` : enveloppe machine, review dans `.result` ;
   - hermétisme : zéro MCP (`--strict-mcp-config`), zéro tool côté devil
     (`--tools ""` ou `--disallowedTools`, au premier qui marche),
     neutralisation des hooks/settings user si un flag le permet ; sinon on
     documente l'héritage au lieu de le nier ;
   - cwd = tmp dir isolé (pas de CLAUDE.md projet parasite).
4. Timeout Bash explicite 540000 ms. Échec ou sortie vide : 1 retry.
5. Parsing : `jq .result` → strip d'éventuelles fences markdown → validation
   des clés requises (score, verdict, summary, criteria, issues) → enveloppe.
6. Nettoyage tmp via trash.

Pas de préflight ollama, pas de vérification de présence du modèle
(contrainte dure, cf. brainstorming) : tout échec runtime est capturé par
l'enveloppe error.

## 4. Skill /devil-spec (v2)

Flow v1 conservé : auto-detect dans `.specs/` (brainstorming.md +
`*-technique.md` ou plan.md), confirmation avant lancement, spawn en
background, rapport formaté (score, critères, tableau d'issues), next steps,
boucle de correction (max 2 re-reviews, issues low ignorées, brainstorm
jamais modifié).

Deltas :
- Argument devil : `/devil-spec [brainstorm] [specs] [gemini|glm|deepseek]`,
  défaut `gemini`. Dispatch vers l'agent correspondant.
- Résolution et passage de SCHEMA_FILE / MISSION_FILE absolus.
- Verdict 3 états : cas `reject` ajouté au rapport → proposer un retour au
  brainstorm plutôt qu'une correction incrémentale.
- Le rapport affiche quel devil a jugé (devil + modèle).
- `subagent_type` : nom plugin-qualifié (`devil:devil-spec-gemini`) ; forme
  exacte vérifiée au build sur le plugin installé.

## 5. Skill /devil-spec-swarm

1. Détection + confirmation identiques à /devil-spec.
2. Spawn des 3 agents DANS UN SEUL message (parallélisme natif du harness).
3. Collecte des 3 enveloppes. Voix absentes (`status: error`) annoncées,
   jamais masquées : 2 voix valides → on continue en le disant ; ≤ 1 voix →
   abort avec rapport d'erreur, pas de verdict.
4. Synthèse par l'orchestrateur, codifiée dans la skill :

   | Situation | Verdict final |
   |---|---|
   | ≥ 2 approve et 0 reject | **VALABLE** |
   | ≥ 2 reject | **JETABLE** |
   | tout le reste | **MODIFICATIONS REQUISES** |

   - Tableau critères : scores des 3 devils côte à côte + moyenne.
   - Issues consolidées : dédup sémantique par l'orchestrateur (même problème
     ≠ mêmes mots), badge de convergence 3/3, 2/3, 1/3, tri par convergence
     puis sévérité, suggestion la plus actionnable retenue.
   - Voix dissonante : un devil qui s'écarte des deux autres (ex. reject
     isolé) a sa section dédiée avec son argument principal. La majorité ne
     l'écrase jamais silencieusement.
5. Rapport final argumenté + next steps :
   - VALABLE → continuer le flow (writing-plans).
   - MODIFICATIONS REQUISES → proposer correction (mêmes règles que
     /devil-spec, issues convergentes d'abord), puis option re-swarm (1 max).
   - JETABLE → proposer le retour au brainstorm ; pas de correction
     incrémentale.

## 6. Marketplace et bascule

- `erom-marketplace/.claude-plugin/marketplace.json` : entrée
  `{ "name": "devil", "source": { "source": "url", "url":
  "https://github.com/eRom/erom-agence-devil.git" }, "description": …,
  "version": "0.1.0", "strict": true }` + bump `metadata.version` 0.4.0 → 0.5.0.
- Commits séparés : erom-agence-devil (build), erom-marketplace (entrée).
- Push GitHub des 2 repos : requis pour l'install par URL. Validation Romain.
- Bascule : après install + validation du plugin, retrait des versions
  user-level (`~/.claude/agents/devil-spec-reviewer.md`,
  `~/.claude/skills/devil-spec/`) via trash pour éviter la collision
  `/devil-spec`. Validation Romain, jamais automatique.

## 7. Vérification

1. Fixtures `examples/` : brainstorming + specs courts, réalistes, noms
   neutres (leçon playbook : pas de `test-*`), avec une dérive volontaire
   pour donner aux devils quelque chose à mordre.
2. Smoke par devil : GLM puis Deepseek sur les fixtures → enveloppe ok +
   review conforme au schéma v2. Gemini : re-run sur les fixtures pour
   non-régression du renommage.
3. Swarm complet sur les fixtures : 3 voix, synthèse, verdict.
4. Dogfood final : `/devil-spec-swarm` sur CETTE spec
   (`.specs/plugin-devil/brainstorming.md` + `architecture-technique.md`).
5. Cas d'erreur : un run avec un modèle volontairement inexistant
   (ex. `glm-typo:cloud`) pour vérifier l'enveloppe error et le comportement
   « voix absente » du swarm. Ceci teste NOTRE gestion d'erreur, pas la
   connectivité validée par Romain.

## 8. Risques et limites assumées

- **Latence** : 3 devils en parallèle ≈ le plus lent (minutes en effort max).
  Assumé : une review swarm est un gate de qualité, pas un lint.
- **Dédup sémantique** de la synthèse = jugement de l'orchestrateur, pas un
  algorithme. Assumé et voulu : c'est son travail de synthèse.
- **Hermétisme claude -p** : si aucun flag ne neutralise hooks/settings user,
  on documente l'héritage, on ne le nie pas. Décision figée au build.
- **Confidentialité** : les devils cloud passent par ollama.com ; le contenu
  des specs part chez un tiers. Même statut que Gemini via agy, assumé.
- **Coûts/quota ollama cloud** : hors périmètre, gérés par Romain.
