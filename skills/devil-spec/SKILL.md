---
name: devil-spec
description: "Review critique de specs tech par un avocat du diable au choix (Gemini par défaut, GLM, Deepseek). Compare specs au brainstorm pour dérives/manques/incohérences. Triggers: /devil-spec, 'contre spec', 'review spec', 'critique les specs'."
user-invocable: true
allowed-tools: Read, Glob, Grep, Agent, AskUserQuestion, Edit
---

# /devil-spec — Review critique de specs par un avocat du diable

Un devil externe (Gemini via agy, ou GLM/Deepseek via claude CLI sur ollama
cloud) juge des specs techniques contre leur brainstorm d'origine. L'agent
wrapper gère l'appel et le parsing ; toi (l'orchestrateur) tu présentes le
rapport et guides les corrections.

## Syntaxe

```
/devil-spec                                    # auto-detect, devil gemini
/devil-spec glm                                # auto-detect, devil glm
/devil-spec brainstorm.md specs.md             # paths explicites, gemini
/devil-spec brainstorm.md specs.md deepseek    # paths + devil
```

## Étape 0 — Résoudre le devil et les chemins du plugin

- Devil : le dernier argument s'il vaut `gemini`, `glm` ou `deepseek` ; sinon
  `gemini` par défaut.
- Racine du plugin : deux niveaux au-dessus du « Base directory for this
  skill » injecté ci-dessus. Résous en absolu :
  - `SCHEMA_FILE` = `<racine>/scripts/devil-spec-schema.json`
  - `MISSION_FILE` = `<racine>/scripts/devil-spec-mission.md`
- Vérifie que les deux fichiers existent (Read). S'ils manquent, arrête et
  signale un plugin corrompu.

## Étape 1 — Identifier les fichiers d'entrée

### Si les paths sont fournis en argument
Utilise-les directement. Vérifie qu'ils existent.

### Si aucun path
Auto-detect :
1. Cherche `brainstorming.md` et `*-technique.md` ou `plan.md` dans `.specs/`
   (Glob `**/*.md`)
2. Si plusieurs chantiers existent, demande à Romain lequel reviewer
3. Si un seul chantier, prends le brainstorming.md + le fichier de specs le
   plus récent

### Confirmation
Toujours confirmer avant de lancer :

> **Fichiers détectés :**
> - Brainstorm : `.specs/mvp/brainstorming.md`
> - Specs : `.specs/mvp/architecture-technique.md`
> - Devil : glm (glm-5.2:cloud)
>
> Je lance la review ?

Attends la confirmation de Romain (« oui », « go », « lance »).

## Étape 2 — Lancer le sous-agent

Spawn l'agent du devil choisi — `devil:devil-gemini`, `devil:devil-glm` ou
`devil:devil-deepseek` ; si ce type est introuvable (plugin non chargé),
retente sans préfixe, ex. `devil-glm` :

```
Agent(
  subagent_type: "devil:devil-<devil>",
  prompt: "MISSION_FILE=<abs>\nSCHEMA_FILE=<abs>\nVALIDATE_JQ=has(\"score\") and has(\"verdict\") and has(\"summary\") and has(\"criteria\") and has(\"issues\") and (.verdict | IN(\"approve\",\"rework\",\"reject\"))\nINPUTS:\nBRAINSTORMING:<abs brainstorm>\nSPECS:<abs specs>\n\nExécute la procédure de transport."
)
```

Annonce avant le spawn : « **Review en cours…** <devil> analyse les specs vs
le brainstorm (jusqu'à 9 min). »

## Étape 3 — Parser l'enveloppe et présenter le rapport

Le retour de l'agent est UNE ligne JSON : `{devil, model, status,
review|error+detail}`.

### Si `status: "error"`
```
══════ REVIEW ÉCHOUÉE ══════

Devil : <devil> (<model>)
Erreur : <error> — <detail>

→ Relance (/devil-spec <devil>), ou essaie un autre devil
  (/devil-spec glm|deepseek|gemini), ou review manuelle.
```

### Si `status: "ok"`
```
══════ REVIEW DE SPECS ══════

Devil : <devil> (<model>)
Score : <score>/100  [APPROVE ✓ | REWORK ✗ | REJECT ☠]

Critères :
  Fidélité      <n>/100  <commentaire court>
  Complétude    <n>/100  <commentaire court>
  Cohérence     <n>/100  <commentaire court>
  Faisabilité   <n>/100  <commentaire court>
  Sécurité      <n>/100  <commentaire court>
  Clarté        <n>/100  <commentaire court>

Résumé : <summary>
```

Issues (si non vide), tableau trié par sévérité (critical > high > medium > low) :

```
| Sév | Cat | Problème | Suggestion | Source |
```

## Étape 4 — Next steps selon le verdict

### approve (score ≥ 80)
> **Verdict : APPROVE** — Les specs sont solides. [issues mineures s'il y en a]
> → On continue le flow ?

### rework (50-79)
> **Verdict : REWORK** — <n> issues à adresser.
> 1. **Corriger** — j'adresse les issues dans les specs
> 2. **Ignorer** — on passe quand même (tu assumes)
> 3. **Re-review** — je corrige puis je relance le même devil

### reject (< 50)
> **Verdict : REJECT** — la spec trahit le brainstorm ou repose sur des choix
> indéfendables. Pas de correction incrémentale : on rouvre le brainstorm
> (réviser l'intention, ou retailler le périmètre), puis on réécrit les specs.

Attends la décision de Romain.

## Étape 5 — Correction (si demandée)

1. Lis le fichier de specs complet
2. Adresse chaque issue par sévérité décroissante (critical > high > medium)
3. Explique brièvement chaque changement
4. Écris via Edit — UNIQUEMENT le fichier de specs
5. Résumé des corrections, puis :
   - **Re-review** → relance le MÊME devil, mêmes fichiers → Étape 3
   - **Corriger** seul → demande validation à Romain

## Règles

- Ne JAMAIS modifier le brainstorm — seulement les specs.
- Ne JAMAIS inventer du contenu — corriger sur la base des issues + du brainstorm.
- Les issues `low` ne sont PAS corrigées sauf demande explicite.
- Maximum 2 cycles de re-review ; après 2 rework consécutifs, Romain tranche.
- Le devil ne modifie rien : c'est toi qui corriges.
