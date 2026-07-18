# Fichiers clés — erom-agence-devil

> MàJ : 2026-07-18

## Manifest
- `.claude-plugin/plugin.json` — name `devil`, `"skills": "./skills/"`. PAS de
  clé `agents` (rejetée par le schéma ; agents auto-découverts depuis `agents/`).

## Contrat commun (scripts/)
- `scripts/spec-review-schema.json` — schéma review : score 0-100, verdict
  `approve|rework|reject`, 6 critères scorés+commentés, issues typées.
- `scripts/devil-mission.md` — mission partagée (architecte senior, 6 critères,
  seuils de verdict). Un seul endroit à éditer pour durcir les 3 devils.

## Agents (agents/) — wrappers Sonnet, color red, tools Bash/Read/Glob/Grep
- `devil-spec-gemini.md` — appelle agy ; assemble prompt (chemins de fichiers) ;
  agy écrit review.json ; timeout 540000 + 1 retry ; enveloppe.
- `devil-spec-glm.md` — claude -p ollama (glm-5.2:cloud) ; contenus embarqués ;
  parse `.result` + strip fences ; enveloppe.
- `devil-spec-deepseek.md` — DÉRIVÉ de glm par sed (modèle d'abord). À
  régénérer via sed si glm change, pour rester en phase.

## Skills (skills/)
- `devil-spec/SKILL.md` — unitaire ; arg devil (gemini défaut) ; auto-detect
  `.specs/` ; rapport ; boucle correction (max 2 re-review).
- `devil-spec-swarm/SKILL.md` — 3 agents en UN SEUL message (parallèle) ;
  quorum tolérant aux voix absentes ; synthèse VALABLE/MODIFICATIONS/JETABLE.

## Fixtures & specs
- `examples/brainstorming.md` + `examples/specs.md` — couple veilleur, 6 dérives
  volontaires (SaaS, OpenAI, clé en clair, SQLite vs JSON, export manquant, ni
  tests ni erreurs). Oracle des smoke tests, non annotées.
- `.specs/plugin-devil/{brainstorming,architecture-technique,plan}.md` — design
  validé du plugin (jugé MODIFICATIONS REQUISES par son propre swarm).

## Marketplace (repo séparé)
- `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json` —
  entrée `devil` (source URL github), metadata.version 0.5.0.
