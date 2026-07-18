# Fichiers clés — erom-agence-devil

> MàJ : 2026-07-18 (v0.2.0)

## Manifest
- `.claude-plugin/plugin.json` — name `devil`, version 0.2.0, `"skills":
  "./skills/"`. PAS de clé `agents` (voir gotchas.md « Manifest de plugin »).

## Contrats par exercice (scripts/)
- `devil-spec-mission.md` + `devil-spec-schema.json` — exercice spec :
  6 critères scorés, verdict approve|rework|reject, issues typées.
- `devil-brain-mission.md` + `devil-brain-schema.json` — exercice brain :
  assessment (1 ligne non évaluative) + questions 0..5 {question, domain,
  risk, criticality blocking|important|exploratory}, maxItems 5.

## Agents (agents/) — transport pur, Sonnet, color red, tools Bash/Read/Glob/Grep
- `devil-gemini.md` — agy ; inputs passés en CHEMINS (agy lit) ; review lue
  depuis fichier écrit par agy ; timeout 540000 + 1 retry ; enveloppe.
- `devil-glm.md` — claude -p ollama ; inputs EMBARQUÉS (run hermétique) entre
  marqueurs `=== BEGIN LABEL ===` ; parse .result + strip fences.
- `devil-deepseek.md` — jumeau sed de devil-glm (modèle d'abord). À régénérer
  par sed si glm change + contrôle post-gen (0 « glm » résiduel).
- Contrat d'entrée commun : MISSION_FILE, SCHEMA_FILE, VALIDATE_JQ (posée en
  single quotes bash), INPUTS lignes `LABEL:abs` → variables IN1_/IN2_.

## Skills (skills/)
- `devil-spec/SKILL.md` — unitaire spec ; 2 inputs ; boucle correction max 2.
- `devil-spec-swarm/SKILL.md` — tribunal ; tri convergence PUIS sévérité.
- `devil-brain/SKILL.md` — unitaire brain ; 1 input ; 0 question = fin sans
  Q&A ; Q&A amende le brainstorming avec les réponses de Romain uniquement.
- `devil-brain-swarm/SKILL.md` — 3 voix parallèles ; consolidation par
  l'orchestrateur (LLM natif) ; tri criticité PUIS convergence (inverse de
  spec-swarm, assumé) ; écartées → non-buts possibles.
- Les 4 : spawn `devil:devil-<nom>` (fallback sans préfixe), chemins résolus
  2 niveaux au-dessus du base dir, VALIDATE_JQ jumeaux byte-identiques
  unitaire/swarm.

## Fixtures & specs
- `examples/brainstorming.md` + `examples/specs.md` — couple veilleur,
  6 dérives plantées. Oracle des smokes (spec → 3/3 reject attendu).
- `.specs/plugin-devil-brain/{brainstorming,architecture-technique,plan}.md`
  — design v0.2.0 complet, avec « Limites connues » (dogfood) et pistes v0.3.

## Ledger de chantier
- `.superpowers/sdd/` — trace complète du chantier v0.2.0 : 7 task-briefs +
  7 reports + 7 diffs de review + progress.md. Point d'entrée pour comprendre
  comment la v0.2.0 a été menée.

## Marketplace (repo séparé)
- `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json` —
  metadata 0.6.0, entrée `devil` version 0.2.0 (URL github).
