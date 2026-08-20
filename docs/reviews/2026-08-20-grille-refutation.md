---
date: 2026-08-20
target: working tree (arbre sale, 5 fichiers suivis)
range: 349dd70 + working tree
devils: deepseek (deepseek-v4-pro:cloud[1m], effort dégradé à low, voir Couverture)
porte: non applicable (le dépôt ne porte ni package.json ni script de vérification)
intent: .claude/notes/recuperable-security-guidance.md
stack: aucun
verdict: GO AVEC RÉSERVES
---

# Review de merge - grille de réfutation (dogfood)

## Verdict : GO AVEC RÉSERVES

Le changement injecte dans la porte de merge la séparation rappel / précision
reprise de `security-guidance` 2.0.7 : polarité de rappel sur l'axe sécurité
côté devil, grille de réfutation à huit motifs côté orchestrateur, périmètre
de vérification élargi à toute issue `security`. Une réserve, Confirmée : le
motif de réfutation « Pré-existant » ne regardait que les lignes `+`, ce qui
aurait réfuté à tort toute garde supprimée. Corrigée dans la foulée (voir
Findings), la réserve est levée.

Run de dogfood, sur son propre diff : la review a été instruite à la main,
la skill n'étant pas installée depuis ce dépôt.

## Couverture

Fichiers reviewés : 5 suivis (+206 / -19), FILES complet (52 751 octets, très
en deçà du budget de 200 Ko). DIFF 19 602 octets, INTENT 15 192 octets.
Paquet total envoyé au modèle : 101 415 octets.

- `_memory_/patterns.md`, `plugin/README.md`,
  `plugin/scripts/devil-review-mission.md`, `plugin/skills/review/SKILL.md`,
  `plugin/skills/review-swarm/SKILL.md`.
- Scan anti-fuite : clean sur les 9 patterns figés, aucun hit.
- Porte déterministe : non applicable, le dépôt n'a ni `package.json` ni
  commande de vérification. Aucune sortie à recopier.

Ce qui n'a PAS été couvert, et doit être lu comme une réserve de méthode :

- **Le devil a tourné à effort `low`, pas `max`.** À `max`, deux tentatives
  de 9 minutes n'ont produit aucun octet. La cause est établie et
  reproductible (voir `_memory_/gotchas.md`, section « Effort max + gros
  paquet ») : ce n'est ni le modèle ni le volume seuls, c'est leur
  combinaison. La critique rendue ici est donc celle d'un devil à effort
  dégradé ; une review à `max` reste due.
- **Une seule voix.** Pas de tribunal, donc aucune convergence pour
  départager. Les angles morts d'un modèle unique ne sont pas couverts.
- **Le fichier non suivi `.claude/settings.local.json.bak` a été écarté du
  paquet**, contre la lettre de l'Étape 1 qui inclut les non suivis. C'est
  un résidu d'outillage produit par `/session-profile` en début de session,
  étranger au changement. Écart assumé et tracé ici.
- **Aucun test automatisable** : le changement porte sur des contrats de
  prompt. La seule validation possible est l'usage réel, ce que ce dogfood
  amorce sans l'épuiser.

## Findings

| Statut | Sév | Cat | Fichier | Problème | Scénario | Preuve |
|---|---|---|---|---|---|---|
| Confirmée | medium (relevée de low) | correctness | `plugin/skills/review/SKILL.md:249` | Le motif de réfutation 1 « Pré-existant » ne testait que l'absence sur les lignes `+`. Une garde supprimée n'apparaît sur aucune ligne `+`. | Un devil signale une garde retirée par le diff ; l'orchestrateur applique le motif 1 à la lettre, ne trouve le code sur aucune ligne `+`, et réfute la régression comme pré-existante. Elle passe la porte de merge. | Lu à `review/SKILL.md:249` : le texte disait « n'apparaît sur aucune ligne `+` du diff ». La forme 2 de la mission (`devil-review-mission.md`, « Régression de contrôle ») demande précisément de chercher ce que des lignes `-` retirent. Contradiction interne au même changement. |

Sévérité relevée de `low` à `medium` par l'orchestrateur : le défaut vise le
mécanisme même qui doit attraper les régressions de contrôle.

Le scénario fourni par le devil illustrait mal sa propre trouvaille : il
décrivait le cas « validateur retiré ET remplacé », où l'issue porte sur le
remplacement, qui est sur une ligne `+`, et où le motif 1 ne s'applique donc
pas. Le cas qui tient est celui de la garde supprimée SANS remplacement.
Finding retenu sur son fond, pas sur son illustration.

Correction appliquée : le motif 1 lit désormais « ni en `+` ni en `-` », avec
la raison écrite en clair dans le texte du motif.

Findings frontière (Claude) : aucun. Des quatre angles du balayage, trois
sont sans objet sur un diff qui ne contient aucun code exécutable (validation
d'entrée vers chemin de fichier, réponse HTTP d'erreur, DX-breakage). Le
quatrième, copy user-visible contre comportement, a été vérifié : les
`description:` de `review` et `review-swarm` et celle de `plugin.json` ne
nomment pas le périmètre de vérification et restent donc exactes. Le README,
qui le nommait, a été mis à jour dans le changement lui-même.

## Annexes

### Réfutées

Aucune.

### Non ancrées

Aucune. L'unique issue pointe `review/SKILL.md:249`, dans la plage modifiée
233-290.

### Critères du devil

Score global 88, verdict `approve`.

- **correctness 90** : « Logique de processus cohérente entre les trois
  fichiers. Une asymétrie de formulation (motif 1 « Pré-existant » vs
  condition de ré-ancre « ligne + ou - ») peut faire réfuter à tort une
  régression de contrôle. »
- **architecture 92** : « La séparation rappel/précision est bien répartie :
  rappel élargi dans la mission devil, filet élargi (périmètre de
  vérification) dans l'Étape 9. Pas de duplication ; la skill swarm
  référence la skill unitaire au lieu de recopier. »
- **security 95** : « La méthode est solide : neuf formes avec conditions de
  non-émission, liste « ne rends PAS » avec l'exception DoS-sur-borne-cassée,
  et grille de réfutation à polarité SURVIE avec preuve exigée. Le rappel
  élargi est compensé par le filet élargi, conformément à l'INTENT. »
- **performance 95** : « Aucun chemin d'exécution modifié. Le coût en tokens
  du prompt sécurité est un arbitrage assumé par l'INTENT (recommandation C),
  pas une régression. »
- **tests 70** : « Aucun test automatisable pour un changement de
  prompt/processus. L'INTENT l'assume explicitement (« À surveiller au
  premier dogfood ») : la validation est différée au premier usage réel. »
- **maintainability 90** : « Texte dense mais lisible, conventions du fichier
  hôte respectées (ancrage file:ligne, motifs nommés, conditions de
  non-émission). La note patterns.md trace la provenance et le point de
  surveillance. »

### Porte déterministe

Non applicable : aucun `package.json` à la racine, aucune commande de
vérification dans ce dépôt. Rien à recopier.

## Ce que le dogfood a appris sur l'outil lui-même

Trois observations, au-delà du verdict sur le diff :

1. **L'anti-bruit de la nouvelle section sécurité tient.** Sur cinq fichiers
   markdown sans une ligne de code exécutable, les neuf formes n'ont produit
   aucune issue `security` parasite. Le critère est scoré 95 avec un
   commentaire argumenté et zéro issue : le devil a compris qu'il n'y avait
   pas de sink, plutôt que de meubler.
2. **L'axe `intent` n'a pas vu ce qu'il devait voir.** L'INTENT fourni décrit
   l'option C ; le changement va au-delà (périmètre de vérification élargi,
   garde de sévérité, README, mémoire projet). Aucune issue `intent` n'a été
   rendue, le devil concluant à un alignement complet. C'est un manque de
   rappel sur cet axe, à recouper sur un prochain run avant d'en tirer une
   règle.
3. **La porte de merge ne peut pas rendre à effort `max` sur un paquet
   réaliste.** Défaut bloquant du plugin, découvert ici, non corrigé :
   l'arbitrage qualité contre latence revient à Romain.
