---
status: proposed
date: 2026-08-18
chantier: plugin-devil-review (erom-devil v0.6.0)
---

# Brainstorming - devil review (plugin erom-devil v0.6.0)

> Validé le 2026-08-18 (session Fable 5, effort max). Décisions issues du
> Q&A Romain x Claude du même jour.
>
> Chantier-gate franchi le 2026-08-18, réponse de Romain verbatim : « Ça sert
> le lancement des projets web-stack-starter. C'est le maillon qui me manque
> pour présenter ma méthode de dev aux futurs clients. »

## Problème

Le gate de merge existe déjà dans le plugin (`code`/`code-swarm`), mais c'est
une critique, pas un process :

- aucune porte déterministe : les devils paient pour ce que tsc et Biome
  disent gratuitement, et une review sur arbre rouge est du bruit ;
- l'intention est un argument optionnel rarement fourni : la review ne sait
  pas ce que le code devait faire (manques et scope creep invisibles) ;
- aucune grille par stack : les pièges spécifiques Next 16 / Prisma 7 du
  starter (DAL, propriété, Server Actions) ne sont vérifiés par personne ;
- les findings ne sont pas vérifiés : leçon spec-swarm (playbook), les devils
  produisent des faux positifs ET ratent des défauts plus graves ; le
  garde-fou sécurité actuel ne peut jamais tomber, même sur un faux positif
  évident ;
- aucune trace persistante : le rapport meurt dans le chat, alors que la
  méthode vendue aux clients a besoin d'un maillon visible et traçable.

## Objectif

`/erom-devil:review` et `/erom-devil:review-swarm` : la porte de merge
complète, avant tout merge vers main. Porte déterministe, chasse à
l'intention, grille par stack, critique devils, vérification contradictoire
par Claude, verdict GO/NO-GO, rapport persistant dans le repo reviewé.

Répartition des rôles actée : `code` reste la critique gratuite et jetable
« à chaque phase » ; `review` est la porte de merge. Les deux vivent.

## Matière analysée (sources du design)

- **`code`/`code-swarm` v0.5.1** : tout l'outillage durci est hérité par
  référence (résolution du target, packaging hermétique, scan anti-fuite,
  ancrage, quorum, consolidation, garde-fou).
- **Corpus d'exemples** (`~/dev/tmp/.claude/skills/code/`) : commons Gemini
  (discipline de sévérité, anti-bruit de ton), skill Matt Pocock (axe spec
  séparé, standards du repo prioritaires, « skip ce que l'outillage
  applique »), orchestrateur Grok (hygiène artefacts, taille de diff),
  persona reviewer Grok (DX-breakages : env vars renommées, ports remappés),
  superpowers requesting-code-review (review contre l'intention, read-only).
- **Patterns built-in Claude Code** : vérification adversariale des findings
  et étiquette Confirmé/Hypothèse (doctrine ReportFindings/ultrareview),
  réutilisés en passe de VÉRIFICATION, pas de découverte.
- **Playbook** : leçon spec-swarm (vérifier avant consolidation), leçon
  portail-CCA (les frontières cross-module survivent aux reviews scopées au
  diff), passe sécu en fin de chantier sur le diff complet, revue finale de
  branche au tier pense.
- **web-stack-starter** (CLAUDE.md + README, vérifiés le 17/08/2026) :
  matière du pack `nextjs.md`.

## Décisions validées

| Sujet | Décision |
|---|---|
| Architecture | 4e exercice du contrat v0.2.0 : 1 mission + 1 schéma + 2 skills, agents transport INCHANGÉS |
| Réutilisation | Étapes de `code` référencées, jamais dupliquées (pattern `code-swarm`) |
| Entrées devil | DIFF + FILES + INTENT + **STACK** (intent et stack optionnels) |
| Porte déterministe | `bun run check` si le script existe, AVANT tout appel modèle ; rouge = STOP ; `no-gates` pour outrepasser, tracé |
| Intention | Chasse systématique : arg > body PR > `.specs/<branche>` > `docs/superpowers/specs/*` > question (« aucune » accepté) |
| Pack stack | `scripts/stacks/nextjs.md` dans le PLUGIN (un seul endroit à maintenir), auto-détecté via package.json, `stack=` en override |
| Passe Claude | Vérification des critical/high (Confirmée / Réfutée / Hypothèse, preuve exigée) + balayage frontières sur 4 angles ; lecture seule stricte |
| Verdict process | GO / GO AVEC RÉSERVES / NO-GO ; les devils gardent leur vocabulaire (matière) |
| Garde-fou sécurité | Amélioré : une critical security Réfutée AVEC PREUVE fait tomber le garde-fou ; Confirmée ou Hypothèse restent bloquantes |
| Rapport | Persistant, `docs/reviews/YYYY-MM-DD-<slug>.md` dans le repo reviewé, en français, écrit par l'orchestrateur |
| Starter | Copie Matt de `.claude/skills/code-review/` trashée, remplacée par une ligne de rituel dans le CLAUDE.md du starter |
| Tier | La skill rappelle en tête : gate de merge, session au tier pense recommandée (xhigh) |

## Battu

- **Process côté starter** (skill dans `.claude/skills/` du template, appelant
  `code`/`code-swarm`) : un fix de process = retoucher N repos clients, drift
  garanti ; la portabilité est déjà acquise, le starter active le plugin.
- **Tout-Claude d'abord** (review native multi-dimensions, devils en
  contre-avis) : brûle le budget protégé à chaque merge, perd la convergence
  multi-familles. Son meilleur élément (vérification adversariale) est
  récupéré en passe de vérification.
- **v2 de `code` au lieu d'une nouvelle paire** : imposer porte, chasse à
  l'intention et rapport à la critique rapide tuerait son côté « gratuit et
  jetable à chaque phase ». Deux rôles, deux noms.
- **Champ `confidence` ou `verification_hint` dans le schéma devil** : la
  confiance auto-déclarée d'un devil est un signal faible ; c'est la passe
  Claude qui étiquette. Le schéma reste une copie de celui de `code`
  (`failure_scenario` suffit comme base de vérification).
- **Lentilles par devil en swarm** (gemini=sécurité, glm=architecture…) :
  détruirait la mesure de convergence, la valeur du tribunal. Les 3 voix
  gardent la mission pleine ; l'angle structurel vit dans la passe Claude.

## Non-buts

- Pas de mode CI/headless en v1 : devils locaux (agy, ollama) + skill
  interactive par design (confirmation, décisions, levées de NO-GO).
- Pas de post de review GitHub inline (PENDING) : parké, le pattern existe
  dans l'exemple Grok si un client le demande un jour.
- Pas de nouveau devil, aucun changement de transport ni d'agent.
- Pas de review par tâche pendant le dev : territoire superpowers, assumé.
- Pas de seuil de coverage ni de score consolidé recalculé.
- Non-buts de `code` hérités : pas de review de stock, pas de devils
  outillés sur le repo, pas de `--fix` automatique.

## Points d'attention livraison

- Bump 0.6.0, marketplace via /plugin-release, uninstall + install.
- README plugin : table des exercices + clarification des rôles
  (`code` = critique jetable, `review` = porte de merge).
- `_memory_` du repo à mettre à jour à l'implémentation.
- Starter : trash de `.claude/skills/code-review/` (copie Matt non adaptée,
  référence un issue-tracker inexistant) + ligne de rituel dans la section
  Vérification du CLAUDE.md.
- CLAUDE.md global de Romain : proposer (sans l'appliquer d'office) que la
  ligne passe-sécu mentionne `review-swarm` comme couverture du gate de
  merge, `/security-review` restant l'outil spécialisé sur demande.
