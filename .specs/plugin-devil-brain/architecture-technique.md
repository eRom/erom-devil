# Architecture technique — devil brain (plugin devil v0.2.0)

> Validé le 2026-07-18. Complète `brainstorming.md` (les décisions produit y font foi).

## 1. Arborescence cible

```
agents/devil-gemini.md            renommé + généralisé (ex devil-spec-gemini.md)
agents/devil-glm.md               renommé + généralisé (ex devil-spec-glm.md)
agents/devil-deepseek.md          dérivé de devil-glm.md par sed (inchangé dans l'esprit)
scripts/devil-spec-mission.md     renommé (ex devil-mission.md), contenu inchangé
scripts/devil-spec-schema.json    renommé (ex spec-review-schema.json), contenu inchangé
scripts/devil-brain-mission.md    NOUVEAU : mission socratique
scripts/devil-brain-schema.json   NOUVEAU : schéma questions
skills/devil-spec/SKILL.md        màj : agents renommés, chemins renommés, INPUTS étiquetés
skills/devil-spec-swarm/SKILL.md  màj : idem
skills/devil-brain/SKILL.md       NOUVEAU : unitaire (arg devil, gemini défaut)
skills/devil-brain-swarm/SKILL.md NOUVEAU : 3 voix parallèles + consolidation
```

Versioning : `plugin.json` → 0.2.0 ; marketplace bump ; réinstall après push.

## 2. Agents généralisés (transport pur)

L'agent ne connaît plus l'exercice. Il reçoit dans son prompt de spawn :

- `MISSION_FILE` : chemin absolu de la mission.
- `SCHEMA_FILE` : chemin absolu du schéma JSON attendu.
- `INPUTS` : liste ordonnée de lignes `LABEL:CHEMIN_ABSOLU` (1 à N fichiers).

Assemblage du prompt modèle : contenu de la mission, puis chaque input sous un
heading `## LABEL`, puis l'exigence de sortie JSON strictement conforme au
schéma (embarqué). L'insertion des inputs suit le transport, comme en v0.1.0 :
contenus embarqués pour glm/deepseek (run hermétique sans accès disque),
chemins de fichiers pour agy (qui lit lui-même). La mécanique de transport
reste au millimètre celle de v0.1.0 :

- **gemini (agy)** : `--print` dernier flag, `< /dev/null`, review lue depuis
  le fichier écrit par agy (bug stdout #76), timeout Bash 540000 ms + 1 retry.
- **glm / deepseek (claude -p → ollama cloud)** : env
  `ANTHROPIC_AUTH_TOKEN=ollama ANTHROPIC_BASE_URL=http://localhost:11434
  ANTHROPIC_API_KEY="" CLAUDE_CODE_EFFORT_LEVEL=max`, flags
  `--dangerously-skip-permissions --strict-mcp-config --tools ""
  --setting-sources "" --no-session-persistence -p --output-format json`,
  parsing `.result` + strip fences, detail d'erreur depuis
  `[.api_error_status] + .result` (jamais stderr). Timeout + retry idem.
- Enveloppe de retour inchangée :
  `{"devil": "...", "model": "...", "status": "ok", "review": {…}}` ou
  `{"devil": "...", "model": "...", "status": "error", "error":
  "CLI_FAILED|PARSE_ERROR|TIMEOUT", "detail": "..."}`.
- deepseek régénéré depuis glm :
  `sed -e 's/glm-5\.2:cloud/deepseek-v4-pro:cloud/g' -e 's/glm/deepseek/g'
  -e 's/GLM/Deepseek/g'` (modèle d'abord).

Appels par exercice :

| Skill | MISSION_FILE | SCHEMA_FILE | INPUTS |
|---|---|---|---|
| devil-spec(-swarm) | devil-spec-mission.md | devil-spec-schema.json | `BRAINSTORMING:…` + `SPECS:…` |
| devil-brain(-swarm) | devil-brain-mission.md | devil-brain-schema.json | `BRAINSTORMING:…` |

## 3. Contrat brain

### Mission (`devil-brain-mission.md`)

Rôle : architecte senior en sparring partner socratique. Entrée : un doc de
brainstorming (définition de besoins). Sortie : les questions les PLUS
DANGEREUSES jamais posées dans ce doc, 5 maximum, en français.

Grille aide-mémoire (à balayer mentalement, JAMAIS à réciter) : périmètre et
non-buts, utilisateurs et parcours, données et cycle de vie, sécurité et
accès, erreurs et modes dégradés, exploitation/ops, coûts et limites,
dépendances externes, réglementaire.

Interdictions dures :
- Ne poser une question QUE si elle est spécifique à CE projet et que
  l'absence de réponse crée un risque réel et nommé.
- Une question dont la réponse figure déjà dans le doc = faute.
- Aucune question générique valable pour n'importe quel projet.
- 0 question est une sortie légitime si le doc est solide.

### Schéma (`devil-brain-schema.json`)

```json
{
  "assessment": "impression générale en une ligne (toujours présente)",
  "questions": [
    {
      "question": "formulée pour être posée telle quelle en session",
      "domain": "domaine libre, pas d'enum (la grille n'est qu'un aide-mémoire)",
      "risk": "ce qui se passe si on écrit la spec sans y répondre",
      "criticality": "blocking | important | exploratory"
    }
  ]
}
```

`questions` : 0 à 5 items (maxItems 5). Valeurs d'enum en anglais (cohérence
avec `approve|rework|reject` du schéma spec), contenus en français.

## 4. Skills brain

### `/devil-brain [devil] [fichier]` (unitaire)

1. Résolution : devil (défaut gemini), doc (arg explicite >
   `.specs/*/brainstorming.md` le plus récent > demander).
2. Chemins mission/schéma résolus en absolu depuis la racine plugin (deux
   niveaux au-dessus du base dir, comme v0.1.0).
3. Spawn `devil:devil-<devil>` (fallback sans préfixe) avec le contrat § 2.
4. Restitution : tableau trié (criticité), avec risque par question, puis
   Q&A ciblé : Romain écarte, on traite les retenues une par une, le doc de
   brainstorming est amendé au fil de l'eau.
5. Si `questions` est vide : « rien de dangereux à signaler, prêt à
   spécifier » + assessment. Pas de Q&A forcé.

### `/devil-brain-swarm [fichier]` (les 3 voix)

1. Résolution du doc, puis spawn des 3 agents en UN SEUL message (parallèle).
2. Tolérance aux voix absentes comme spec-swarm : 2 voix = OK annoncé,
   1 voix = dégradé annoncé, 0 voix = échec franc.
3. Consolidation : regroupement SÉMANTIQUE des questions équivalentes (même
   angle mort formulé différemment), badge convergence 3/3, 2/3, 1/3. La
   criticité retenue pour un groupe = la plus haute parmi les voix groupées.
4. **Tri : criticité PUIS convergence** (différence assumée avec spec-swarm :
   une bloquante 1/3 vaut plus qu'une exploratoire 3/3 ; la convergence
   départage à criticité égale).
5. Restitution identique à l'unitaire (tableau consolidé avec convergence,
   écarte/retiens, Q&A, doc amendé, écartées → non-buts possibles).
6. Toutes voix à 0 question : « rien à signaler, spécifie » + les
   assessments. Pas de Q&A.

### En cours de session (second moment d'usage)

Aucune mécanique dédiée : si le doc n'existe pas encore, Claude fige d'abord
un draft (fichier réel dans `.specs/<chantier>/brainstorming.md`), puis
invoque la skill normalement. Les questions retenues nourrissent la suite du
Q&A de brainstorming.

## 5. Migration devil-spec (refactor interne, surfaces intactes)

- `skills/devil-spec/SKILL.md` + `skills/devil-spec-swarm/SKILL.md` : spawn
  `devil:devil-{gemini,glm,deepseek}`, chemins `devil-spec-mission.md` /
  `devil-spec-schema.json`, INPUTS `BRAINSTORMING:` + `SPECS:`.
- Aucun changement de verdicts, seuils, synthèse, boucle de correction.
- Grep final : zéro référence restante à `devil-spec-gemini|glm|deepseek`,
  `devil-mission.md`, `spec-review-schema.json`, zéro chemin `~/.claude/` en dur.

## 6. Vérification (pas de suite de tests classique)

1. **Non-régression spec** : `/devil-spec-swarm` sur les fixtures veilleur →
   doit converger `reject` et attraper les 6 défauts plantés.
2. **Smoke brain unitaire** (glm, le moins cher à itérer) sur
   `examples/brainstorming.md` seul → questions spécifiques veilleur,
   0 récitation de grille, ≤ 5.
3. **Chemin d'erreur** : modèle bidon → enveloppe `status:"error"` avec detail
   `[404] …`.
4. **Inventaire** : `claude --plugin-dir . plugin details devil` → 4 skills +
   3 agents.
5. **Dogfood méta** : `/devil-brain-swarm` sur
   `.specs/plugin-devil-brain/brainstorming.md` lui-même.
6. Bump 0.2.0, marketplace update, push, réinstall, smoke via plugin installé.
