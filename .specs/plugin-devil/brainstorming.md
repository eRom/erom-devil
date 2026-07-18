# Brainstorming — Plugin « devil » (avocats du diable)

> Capture du brainstorm du 2026-07-18 (session Claude + décisions Romain).
> Source de vérité amont pour `architecture-technique.md`. Ne pas modifier après
> validation : les reviews devil comparent les specs à CE document.

## Intention

Un plugin Claude Code (marketplace eRom) qui fournit des « avocats du diable » :
des reviewers critiques externes qui jugent une spec technique par rapport à son
brainstorm d'origine, pour détecter dérives, manques et incohérences AVANT
implémentation.

Le demandeur est généralement Claude lui-même (flow spec → review → décision) :
les réponses doivent donc être strictement parseables.

## Existant (importé)

- Agent `devil-spec-reviewer` (`~/.claude/agents/`) : wrapper Sonnet qui appelle
  Antigravity CLI (agy → Gemini 3.5 Flash High), fait écrire la review JSON dans
  un fichier tmp (bug stdout agy #76), 1 retry, nettoyage via trash.
- Skill `devil-spec` (`~/.claude/skills/`) : auto-detect des fichiers dans
  `.specs/`, confirmation, spawn du sous-agent en background, rapport formaté,
  boucle de correction (max 2 re-reviews, issues low ignorées).
- Schéma JSON : score 0-100, verdict approve/rework, 6 critères (fidelity,
  completeness, consistency, feasibility, security, clarity), issues typées
  severity/category/description/suggestion/source.
- Import verbatim dans ce repo : commit `e27c4f3`.

## Objectifs

1. Choix entre 3 avocats du diable, unitairement : Gemini | GLM | Deepseek.
2. Format de réponse identique pour les 3, parseable par le demandeur.
3. Un « devil review swarm » : orchestration des 3 devils en parallèle.
4. Un résultat final argumenté : spec valable | modifications requises | jetable.

Entrées : toujours 2 fichiers, brainstorming + specs.

## Contrainte dure — harness GLM / Deepseek (validé par Romain)

Les deux modèles tournent via le harness Claude Code pointé sur ollama local,
modèles cloud ollama.com. Romain a testé les deux lignes, elles fonctionnent :

```bash
ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434 \
ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max \
claude --model glm-5.2:cloud --dangerously-skip-permissions -p "..."
# idem avec --model deepseek-v4-pro:cloud
```

- PAS de `ollama pull` : les modèles `:cloud` sont utilisables directement.
- `/api/tags` ne les liste PAS : aucun préflight de présence du modèle,
  ce serait un faux négatif.
- Ne pas re-tester la connectivité de base : acquise. Seuls les flags AJOUTÉS
  par nous (parsing, hermétisme) se valident au build.
- Toutes les options de la ligne de commande `claude` sont disponibles.

## Décisions prises (Q&A du 2026-07-18)

1. **Surface** : `/devil-spec` (unitaire, arg devil, défaut gemini) +
   `/devil-spec-swarm` (les 3 en parallèle + synthèse). Nommage du swarm
   choisi par Romain.
2. **Contrat** : verdict à 3 états chez CHAQUE devil : approve (≥80) /
   rework (50-79) / reject (<50). Le swarm dérive valable / modifications
   requises / jetable.
3. **Architecture** : trio d'agents symétriques ; GLM/Deepseek en texte pur
   (contenus des fichiers embarqués dans le prompt, JSON sur stdout), zéro tool
   côté devil. Swarm = 3 Agent calls parallèles dans un seul message + synthèse
   par l'orchestrateur. Alternatives écartées : agent runner unique paramétré
   (fourre-tout à 2 protocoles), Workflow tool (artillerie pour 3 appels fixes).
4. **Nommage** : plugin `devil`, agents `devil-spec-gemini|glm|deepseek`
   (préfixe qui laisse la place à un futur `devil-code-*`).

## Hors scope (pour plus tard)

- `devil-code` (review de code multi-devils) : hors périmètre de ce chantier.
- Ajout d'autres devils : le design doit juste ne pas l'empêcher.
