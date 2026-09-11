---
date: 2026-08-18
target: branche feat/devil-review vs main
range: 87efb5b..66f1f32
devils: gemini + glm (deepseek absent)
porte: aucune commande
intent: .specs/plugin-devil-review/architecture-technique.md
stack: aucun
verdict: GO AVEC RÉSERVES
---

# Review de merge - feat/devil-review

## Verdict : GO AVEC RÉSERVES

Aucune issue `critical` ni `high` sur les deux voix exprimées. Une issue
`medium` de catégorie `intent` est ancrée et confirmée : le wording de
`no-gates` diverge de la section 4 de la spec. Cette divergence est
délibérée et tracée (elle rend le rapport plus honnête en refusant qu'une
porte ROUGE s'enregistre comme porte SAUTÉE), mais elle laisse la spec et
l'implémentation en désaccord littéral : c'est la réserve. Trois défauts de
cohérence documentaire, tous `low`, complètent le dossier.

## Couverture

Diff reviewé : 15 fichiers, +2308 -10, sur `87efb5b..66f1f32`, pathspec
`':(exclude)docs/reviews/'`. FILES complet (137 Ko, sous le budget de
200 Ko), aucun fichier exclu. INTENT fourni. Scan anti-fuite : clean sur les
trois inputs, aucun hit des exclusions dures ni des regex.

**Voix absente : deepseek.** Son appel a échoué en `CLI_FAILED` sur un
plafond dur : « Claude's response exceeded the 32000 output token maximum ».
Cause certaine, pas une hypothèse, l'API la nomme elle-même. Le tribunal a
donc statué à 2 voix sur 3, conformément à la règle de quorum. La grille de
convergence de ce rapport se lit SUR LES VOIX EXPRIMÉES : 2/2 et 1/2, pas
3/3.

**Porte déterministe : aucune commande.** Ce dépôt est en Markdown et JSON
pur, sans `package.json` ni suite de tests. La question a été posée une fois
et tranchée par Romain. Aucune porte n'a donc filtré quoi que ce soit avant
les modèles.

**Ce qui n'a PAS été couvert.** Le périmètre de la passe de vérification
(`critical`, `high`, et `medium` convergentes) s'est trouvé VIDE : il n'y a
ni critical ni high, et l'unique medium n'est portée que par une voix sur
deux. Les quatre issues devils sortent donc en « Non vérifiée » au sens
strict de la procédure. Je les ai vérifiées quand même, une par une, contre
le code réel : le détail est en colonne Preuve. Aucun test n'a été relancé,
il n'y en a pas dans ce dépôt.

**Biais structurel de ce run, à connaître.** C'est un dogfood méta : la
branche reviewée CONTIENT son propre document d'intention (les trois
fichiers `.specs/plugin-devil-review/` sont dans le diff). L'axe `intent`
est donc partiellement tautologique ici, et le résultat sur cet axe ne se
généralise pas à une review ordinaire.

**Substitution de caractères.** Le matériau recopié des devils a été
normalisé sur un seul point : les tirets cadratins sont remplacés par des
tirets simples, exigence du hook du dépôt. Rien d'autre n'a été modifié. Une
coquille de GLM (« amaut » pour « amont ») est laissée telle quelle dans les
citations, signalée ici plutôt que corrigée en silence.

## Findings

| Statut | Conv | Sév | Cat | Fichier | Problème | Scénario | Preuve |
|---|---|---|---|---|---|---|---|
| Confirmée | 1/2 | medium | intent | `plugin/skills/review/SKILL.md:81` | Le wording de `no-gates` diverge de la spec section 4, qui exige « porte : SAUTÉE sur décision » ; la skill écrit « porte : ROUGE, outrepassée sur décision » et fait tourner la porte au lieu de la sauter | Romain lit la spec, attend « SAUTÉE » au rapport, lit « ROUGE, outrepassée » : confusion sur ce que `no-gates` fait réellement | Vérifié : la divergence est réelle et volontaire, tracée comme ruling I-3 au ledger du chantier. La spec dit « saute la porte », la skill la fait tourner et n'empêche que l'arrêt sur rouge |
| Confirmée (frontière, Claude) | n/a | low | maintainability | `_memory_/architecture.md:62` | La ligne `examples/` de l'arborescence décrit « fixtures veilleur (6 défauts) + code (5 défauts + secret) » et ignore la fixture stack livrée par ce diff | Un exécutant cherche la fixture de la grille nextjs dans l'arborescence de référence et ne la trouve pas | `ls examples/` rend 7 fichiers dont `review-stack.patch` et `review-stack-files.txt`, absents de la ligne |
| Confirmée | 1/2 | low | maintainability | `_memory_/architecture.md:8` | La ligne d'objectif garde « documents amaut, AVANT implémentation » alors que le 4e exercice est une porte de merge post-implémentation ; le README a été corrigé dans ce même diff, pas la mémoire | Un exécutant lit la mémoire, voit « AVANT implémentation » et cherche `review` du côté des documents amont | `sed -n '8,9p'` rend bien « AVANT implémentation. Quatre exercices : », et `README.md:5` dit « du document amont à la porte de merge ». La même fausseté a été corrigée dans un fichier et pas dans l'autre |
| Confirmée | 1/2 | low | correctness | `plugin/skills/review/SKILL.md:93` | Le fichier temp du body de PR vit hors de `TMP_DIR` et échappe au `trash "$TMP_DIR"` de clôture | Chaque run en mode PR laisse un fichier orphelin qui n'est nettoyé que par l'OS, plusieurs jours plus tard sur macOS | Lu : l'Étape 3 crée un `mktemp` propre parce que `TMP_DIR` n'existe qu'à l'Étape 4, et aucune règle ne le rattrape |
| Non ancrée, confirmée | 1/2 | low | maintainability | `_memory_/patterns.md:3` | Le header `> MàJ :` reste à `2026-07-18 (v0.3.0)` alors que les deux autres fichiers mémoire passent à `2026-08-18 (v0.7.0)` dans ce même diff | Au prochain chantier, un exécutant suppose que les patterns review ajoutés en bas de fichier datent de v0.3.0 | `grep -m1 'MàJ' _memory_/*.md` rend 2026-08-18 pour architecture et key-files, 2026-07-18 pour patterns |
| Non ancrée, confirmée (frontière, Claude) | n/a | low | maintainability | `_memory_/architecture.md:63` | Le motif `.specs/plugin-devil{,-brain,-code}/` de l'arborescence ne couvre pas `-review`, et la liste des designs s'arrête à v0.3.0 | La mémoire d'architecture ne mentionne nulle part où vit le design du 4e exercice | `ls -d .specs/*/` rend 4 dossiers, le motif en nomme 3 |
| Non ancrée, confirmée (frontière, Claude) | n/a | low | maintainability | `_memory_/key-files.md:86` | L'entrée marketplace affirme « metadata 0.9.0, entrée erom-devil version 0.5.0 » | Un exécutant se fie à ces versions pour la livraison et croit devoir bumper depuis 0.5.0 | Lu dans `/Users/recarnot/dev/erom-marketplace/.claude-plugin/marketplace.json` : metadata 0.18.3, entrée 0.6.0. Périmé AVANT ce chantier, la livraison le corrigera |

## Annexes

### Réfutées

Aucune. Les sept findings ont tous survécu à la vérification.

### Non ancrées

Trois findings sont hors des plages de hunks et sortent donc du verdict par
l'Étape 8 : `_memory_/patterns.md:3`, `_memory_/architecture.md:63` et
`_memory_/key-files.md:86`. Ils sont VRAIS malgré tout, vérifiés preuve à
l'appui, et leur sévérité `low` fait qu'ils ne bénéficient pas de
l'exception de ré-ancre, réservée aux `critical` et `high`. Le dispositif
écarte donc trois trouvailles justes. C'est défendable, le défaut ne vient
pas du diff, mais c'est une limite à connaître de la porte.

### Voix dissonantes

Écart majeur entre les deux voix exprimées, sur le fond et pas seulement sur
le score. Gemini rend `approve` à 96 avec ZÉRO issue sur un diff de
2 308 lignes ; GLM rend `approve` à 87 avec quatre issues, dont deux
défauts réels de cohérence documentaire que Gemini n'a pas vus.

Le registre de Gemini mérite d'être signalé : ses commentaires de critères
sont en pur éloge, « exemplaire », « sans faille », « irréprochable »,
« Excellente ». La mission dit pourtant en première ligne « Ton travail est
de chercher ce qui cloche dans ce changement, pas de complimenter ». Un
retour à zéro issue est LÉGAL, la mission l'autorise explicitement quand le
changement est excellent. Le registre, lui, est exactement celui qu'elle
interdit. Sur ce dossier, la voix la moins complaisante est la seule qui ait
produit de la valeur.

### Scores devils

| Devil | Modèle | Score | Verdict | correctness | architecture | security | performance | tests | maintainability |
|---|---|---|---|---|---|---|---|---|---|
| gemini | Gemini 3.8 Flash (High) | 96 | approve | 98 | 97 | 98 | 95 | 92 | 96 |
| glm | glm-5.3-flash:cloud[1m] | 87 | approve | 88 | 92 | 93 | 90 | 75 | 82 |
| deepseek | deepseek-v4.1-flash:cloud[1m] | absent | CLI_FAILED | n/a | n/a | n/a | n/a | n/a | n/a |

Moyenne indicative sur les voix exprimées : 91,5. Grille devils : deux
`approve` sur deux voix exprimées, aucun `reject`, donc VALABLE au sens de
la grille de `code-swarm`. Le garde-fou sécurité ne s'arme pas : aucune
issue `critical` de catégorie `security`.

Commentaires par critère, voix par voix, tels que rendus :

- gemini, correctness 98 : « La logique des étapes d'orchestration est sans
  faille : gestion exacte des sources de lecture selon le mode. »
- gemini, architecture 97 : « Excellente réutilisation par référence des
  briques durcies de l'exercice code. »
- gemini, security 98 : « Garde-fou sécurité renforcé avec exigence de
  preuve située pour lever un blocage. »
- gemini, performance 95 : « Spawn parallèle en un seul message pour le
  swarm, budget de 2 minutes par issue pour la passe de vérification. »
- gemini, tests 92 : « Fixtures dédiées créées avec violations de stack
  plantées (Server Action publique, IDOR, Zod 4). »
- gemini, maintainability 96 : « Documentation irréprochable dans les
  skills, README et mémoire du dépôt. »
- glm : scores ci-dessus, ses commentaires détaillés sont repris dans la
  colonne Problème du tableau Findings.

### Porte déterministe

Non lancée : aucune commande. Ce dépôt n'a pas de `package.json` à sa
racine et donc pas de script `check`. La question a été posée une fois à
Romain, conformément au point 2 de l'Étape 2, et la réponse a été « aucune ».
Aucune sortie brute à recopier.
