# Fichiers clés — erom-agence-devil

> MàJ : 2026-07-18 (v0.3.0)

## Manifest
- `.claude-plugin/plugin.json` — name `devil`, version 0.4.0, `"skills":
  "./skills/"`. PAS de clé `agents` (voir gotchas.md « Manifest de plugin »).

## Contrats par exercice (scripts/)
- `devil-spec-mission.md` + `devil-spec-schema.json` — exercice spec :
  6 critères scorés, verdict approve|rework|reject, issues typées.
- `devil-brain-mission.md` + `devil-brain-schema.json` — exercice brain :
  assessment (1 ligne non évaluative) + questions 0..5 {question, domain,
  risk, criticality blocking|important|exploratory}, maxItems 5.
- `devil-code-mission.md` + `devil-code-schema.json` — exercice code :
  6 critères code scorés {score,comment}, verdict approve|rework|reject,
  issues {severity, category (6+intent), file "chemin:ligne",
  failure_scenario obligatoire, suggestion}. VALIDATE_JQ borne scores ET
  critères (testée jq 1.8.1).

## Agents (agents/) — transport pur, Sonnet, color red, tools Bash/Read/Glob/Grep
- `devil-gemini.md` — agy ; inputs passés en CHEMINS (agy lit) ; review lue
  depuis fichier écrit par agy ; timeout 540000 + 1 retry ; enveloppe.
- `devil-glm.md` — claude -p ollama ; inputs EMBARQUÉS (run hermétique) entre
  marqueurs `=== BEGIN LABEL ===` ; parse .result + strip fences.
- `devil-deepseek.md` — jumeau sed de devil-glm (modèle d'abord). À régénérer
  par sed si glm change + contrôle post-gen (0 « glm » résiduel).
- `devil-opus.md` — claude -p (Opus 4.8 xHigh) ; même contrat de transport ;
  hors swarms, dispo en unitaire (`/devil-* opus`).
- `devil-kimi.md` — jumeau sed de devil-glm (kimi-k3:cloud[1m]) ; hors
  swarms comme opus, juge indépendant en unitaire (`/devil-* kimi`).
  Modèle hors forfait Ollama : 402 sans extra usage (voir gotchas.md).
- Les 3 transports ollama portent `[1m]` aux 5 mêmes emplacements (desc,
  enveloppe ok, enveloppe error, ligne d'appel, jq final) et sont
  byte-identiques après normalisation nom+tag. Le tag DOIT être quoté dans
  la ligne d'appel (glob zsh, voir gotchas.md).
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
- `devil-code/SKILL.md` — unitaire code ; résolution target (PR/range/
  ref/auto), packaging TMP_DIR (DIFF jamais tronqué, FILES budget 200 Ko,
  INTENT opt.), scan pré-vol AVANT envoi (STOP sur hit), ancrage
  file:ligne ±3 sur les hunks, correction guidée selon mode (table).
- `devil-code-swarm/SKILL.md` — tribunal code (opus et kimi exclus) ; ancrage par
  voix, consolidation problème de fond, verdict table + garde-fou
  sécurité (critical security ancrée → jamais VALABLE), tri convergence
  puis sévérité.
- Les 6 : spawn `devil:devil-<nom>` (fallback sans préfixe), chemins résolus
  2 niveaux au-dessus du base dir, VALIDATE_JQ jumeaux byte-identiques
  unitaire/swarm.

## Fixtures & specs
- `examples/brainstorming.md` + `examples/specs.md` — couple veilleur,
  6 dérives plantées. Oracle des smokes (spec → 3/3 reject attendu).
- `examples/code-diff.patch` + `code-files.txt` — paquet code planté
  (injection db.ts:17, null deref auth.ts:9, N+1 report.ts:8, dup
  maskName, test sans assertion). `code-secret.patch` — oracle du scan
  pré-vol (AKIA + PRIVATE KEY, exemples doc AWS).
- `.specs/plugin-devil-brain/{brainstorming,architecture-technique,plan}.md`
  — design v0.2.0 complet, avec « Limites connues » (dogfood) et pistes v0.3.

## Ledger de chantier
- `.superpowers/sdd/` — scratch du chantier courant (task-briefs, reports,
  diffs de review, progress.md) ; la trace du chantier v0.2.0 est archivée
  dans `.superpowers/sdd/archive-v0.2.0/`.

## Marketplace (repo séparé)
- `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json` —
  metadata 0.6.1, entrée `devil` version 0.3.0 (URL github).
