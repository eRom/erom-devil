---
name: devil-spec
description: "Review critique de specs tech via Gemini. Compare specs au brainstorm pour dérives/manques/incohérences. Triggers: /devil-spec, 'contre spec', 'review spec', 'critique les specs'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-spec — Review critique de specifications techniques

Review automatisee des specs techniques par Gemini, comparees au brainstorm original.
Le sous-agent `devil-spec-reviewer` (rouge, Sonnet) gere l'appel Gemini et le parsing.
Opus orchestre, presente le rapport et guide les corrections.

## Syntaxe

```
/devil-spec                                    # auto-detect des fichiers
/devil-spec brainstorm.md specs.md             # paths explicites
/devil-spec .specs/mvp/brainstorming.md .specs/mvp/architecture-technique.md
```

## Etape 1 — Identifier les fichiers

### Si les paths sont fournis en argument
Utilise-les directement. Verifie qu'ils existent.

### Si aucun argument
Auto-detect :
1. Cherche les fichiers `brainstorming.md` et `*-technique.md` ou `plan.md` dans `.specs/` (Glob `**/*.md`)
2. Si plusieurs chantiers existent, demande a Romain lequel reviewer
3. Si un seul chantier, prends le brainstorming.md + le fichier de specs le plus recent

### Confirmation
Toujours confirmer les fichiers detectes avant de lancer :

> **Fichiers detectes :**
> - Brainstorm : `.specs/mvp/brainstorming.md`
> - Specs : `.specs/mvp/architecture-technique.md`
>
> Je lance la review ?

Attends la confirmation de Romain (ou un "oui", "go", "lance").

## Etape 2 — Lancer le sous-agent

Spawn le sous-agent `devil-spec-reviewer` (type agent, Sonnet, rouge) :

```
Agent(
  subagent_type: "devil-spec-reviewer",
  model: "sonnet",
  prompt: "BRAINSTORM_FILE=${brainstorm_path_absolu}\nSPECS_FILE=${specs_path_absolu}\n\nExecute la procedure de review.",
  run_in_background: true,
  name: "devil-spec-review"
)
```

Pendant que Gemini tourne, affiche :

> **Review en cours...** Gemini analyse les specs vs le brainstorm.

## Etape 3 — Presenter le rapport

Quand le sous-agent retourne le resultat, formate-le ainsi :

### Si ERROR
```
══════ REVIEW ECHOUEE ══════

Antigravity n'a pas retourne du JSON valide apres 2 tentatives.
Retour brut : [extrait]

→ Tu peux relancer avec /devil-spec ou reviewer manuellement.
```

### Si JSON valide

```
══════ REVIEW DE SPECS ══════

Score : [XX]/100  [APPROVE ✓ | REWORK ✗]

Criteres :
  Fidelite      [XX]/100  [comment court]
  Completude    [XX]/100  [comment court]
  Coherence     [XX]/100  [comment court]
  Faisabilite   [XX]/100  [comment court]
  Securite      [XX]/100  [comment court]
  Clarte        [XX]/100  [comment court]

Resume : [summary]
```

Si des issues existent, les afficher en tableau trie par severite :

```
Issues ([N]) :

| Sev      | Cat          | Probleme           | Suggestion         | Source              |
|----------|--------------|--------------------|--------------------|---------------------|
| critical | fidelity     | [description]      | [suggestion]       | [section]           |
| high     | completeness | [description]      | [suggestion]       | [section]           |
| ...      | ...          | ...                | ...                | ...                 |
```

## Etape 4 — Proposer le next step

### Cas 1 : APPROVE (score >= 80)
> **Verdict : APPROVE** — Les specs sont solides.
> [Mentionner les issues mineures s'il y en a]
>
> → On continue le flow superpowers ?

### Cas 2 : REWORK (score < 80)
> **Verdict : REWORK** — [N] issues a adresser.
>
> Options :
> 1. **Corriger** — Je corrige les specs en adressant les issues
> 2. **Ignorer** — On passe quand meme (tu assumes les risques)
> 3. **Re-review** — Je corrige puis relance Antigravity pour validation

Attends la decision de Romain.

## Etape 5 — Correction (si demandee)

Si Romain choisit "Corriger" ou "Re-review" :

1. Lis le fichier de specs complet
2. Adresse chaque issue par severite decroissante (critical > high > medium)
3. Pour chaque correction, explique brievement ce qui change
4. Ecris le fichier corrige via Edit
5. Affiche un resume des corrections :

```
══════ CORRECTIONS APPLIQUEES ══════

[N] issues adressees :
  - [critical] [description courte] → [ce qui a change]
  - [high] [description courte] → [ce qui a change]
  - ...

Fichier modifie : [path]
```

6. **Si "Re-review"** : relance le sous-agent avec les memes fichiers → retour a l'etape 3
7. **Si "Corriger"** : demande a Romain s'il valide

> Corrections appliquees. On continue le flow ?

## Regles

- Le sous-agent est TOUJOURS lance en **background** (Gemini peut prendre 10-30s)
- Ne JAMAIS modifier le brainstorm — seulement les specs
- Ne JAMAIS inventer du contenu — corriger en se basant sur les issues Antigravity + le brainstorm
- Les issues `low` ne sont PAS corrigees sauf si Romain le demande explicitement
- Maximum 2 cycles de re-review (si 2 rework consecutifs, demander a Romain de trancher)
