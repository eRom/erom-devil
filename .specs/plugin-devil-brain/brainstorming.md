# Brainstorming — devil brain (plugin devil v0.2.0)

> Validé le 2026-07-18. Décisions issues du Q&A Romain × Claude.

## Problème

La chaîne de fabrication est désormais contrôlée en aval : devil-spec garantit
que la spec couvre le brainstorming à ~100 %. Mais le maillon amont reste
aveugle : si le brainstorming lui-même a oublié tout un pan de la solution
(sécurité, features, exploitation…), la spec héritera fidèlement de ces trous.
Le brainstorming est le sommet de la chaîne : il n'existe aucun référentiel
amont contre lequel le juger. C'est un problème d'unknown unknowns.

## Objectif

À la demande, pouvoir s'assurer que la définition des besoins (doc de
brainstorming) est pleinement couverte : aucun pan majeur oublié, aucune
décision structurante jamais posée.

## Thèse

On ne peut pas certifier une complétude absolue (personne ne le peut). On peut
réduire drastiquement la surface d'angles morts avec deux armes :

1. **La diversité des regards** : trois modèles externes aux biais différents
   (Gemini, GLM, Deepseek), la thèse fondatrice du plugin devil.
2. **Des grilles éprouvées** : une taxonomie de domaines comme filet de
   sécurité, sans jamais la réciter.

## Décision fondamentale : l'interrogatoire socratique

devil brain n'est **pas un tribunal**. Contrairement à devil-spec (qui juge une
transformation entre deux artefacts), devil brain est un **sparring partner** :
il rend les questions que le brainstorming n'a jamais posées, formulées pour
être réinjectées directement dans la session de brainstorming.

- **Pas de score, pas de verdict.** On ne note pas contre l'inconnu.
- Le « verdict » émerge en creux : un devil qui rend 0 question dit
  « rien de dangereux, prêt à spécifier ».
- Chaque question porte une criticité (clé de TRI des questions, pas un
  jugement du doc) et chaque devil rend une `assessment` (impression de
  contexte non évaluative, précieuse surtout à 0 question) : ni l'une ni
  l'autre ne notent le doc.

## Décisions validées

| Sujet | Décision |
|---|---|
| Livrable | Questions socratiques uniquement (ni score ni verdict) |
| Entrée | **1 fichier** : le doc de brainstorming |
| Moment | Les deux : gate de fin de brainstorm, OU en cours de session (Claude fige un draft du doc puis appelle) |
| Angles | Mixte : grille aide-mémoire dans la mission + liberté des modèles ; interdiction dure de réciter la grille |
| Budget | **5 questions max par devil**, « tes 5 questions les plus dangereuses » |
| Restitution | Tri puis Q&A ciblé : liste consolidée d'abord, Romain écarte d'un regard, Q&A une par une sur les retenues, doc amendé au fil de l'eau |
| Questions écartées | Peuvent devenir des non-buts explicites dans le doc |
| Architecture | **Généralisation** : trio d'agents transport-only partagés entre devil-spec et devil-brain (pas de duplication) |

## Qualité des questions (le nerf de la guerre)

Le danger n° 1 d'un interrogatoire socratique est le bruit : 30 questions
génériques récitées d'une checklist. Garde-fous :

- Budget dur de 5 : force la sélectivité, le devil tue ses chouchous.
- Question **spécifique à CE projet** uniquement ; son absence de réponse doit
  créer un risque réel et nommé.
- Une question dont la réponse figure déjà dans le doc = faute.
- Chaque question porte son « pourquoi » : le risque couvert si on spécifie
  sans y répondre.

## Flux cible

```
brainstorming ──(devil brain)──> doc enrichi ──> specs ──(devil-spec)──> plan
        ▲                            │
        └────── Q&A sur les ─────────┘
                questions retenues
```

## Non-buts

- Pas de score ni verdict pour devil brain (assumé, choix philosophique).
- Pas de boucle automatique de re-passage : le re-appel après amendement du
  doc est manuel, à la demande.
- Pas de nouveau modèle devil : mêmes trois voix que devil-spec.
- Aucun changement de surface ni de sémantique pour `/devil-spec` et
  `/devil-spec-swarm` (refactor interne uniquement).
- Pas d'appel des devils sur un plan (décision v0.1.0 inchangée : les devils
  voient brainstorming et specs, jamais de secrets).
- Pas d'avertissement de confidentialité à l'appel : les entrées transitent
  par des fournisseurs externes, c'est le principe même d'avocats externes
  (décision v0.1.0, assumée et tracée ici). Corollaire inchangé : jamais de
  secrets dans un brainstorming ou une spec.

## Limites connues (levées par le dogfood, 2026-07-18)

Le plugin s'est interrogé lui-même (`/devil-brain-swarm` sur ce brainstorming).
Ces limites sont assumées, pas résolues : les nommer vaut mieux que les taire.

1. **Le « 0 question » n'est pas calibré.** Rien ne distingue un doc complet
   d'un doc trop mince pour offrir une prise. Le feu vert est structurellement
   biaisé vers les docs les plus à risque. Posture : le 0-question est un
   signal FAIBLE, pas un certificat ; l'`assessment` (toujours présent) porte
   le contexte. Un brainstorming squelettique doit être reconnu comme tel par
   le porteur AVANT d'appeler l'outil.
2. **La diversité des trois voix est postulée, pas prouvée.** Trois modèles
   transformer à corpus chevauchants peuvent partager un angle mort (domaine
   de niche, dimension réglementaire ou sociale, réalité ops d'une petite
   équipe). L'outil réduit la surface d'angles morts, il ne la garantit pas
   nulle : il certifie l'absence de danger DANS l'horizon partagé des modèles,
   pas au-delà.
3. **Le tri est une porte à une personne.** L'auteur écrit le doc ET écarte
   les questions ; rien ne l'empêche d'esquiver précisément ce qui le dérange,
   et un angle mort converti en non-but échappe ensuite à devil-spec (qui
   traite un non-but comme légitimement hors périmètre). Posture : une
   question écartée mérite une ligne de rationale, pour rendre l'esquive
   visible plutôt que silencieuse.
4. **Tension secret ↔ richesse de l'input.** Dé-sensibiliser un brainstorming
   pour l'envoi externe dégrade précisément le spécifique dont brain a besoin
   pour ne pas produire du bruit générique. Pas de filtre local automatique :
   le porteur arbitre au cas par cas entre spécificité et confidentialité, le
   corollaire no-secrets restant la borne dure.
5. **Re-passage sans mémoire.** Sur un doc réinvoqué après amendement, les
   questions déjà écartées réapparaissent (devils stateless). Posture assumée :
   re-appel manuel, le porteur ignore les redites ; une mémoire des écartées
   est hors scope v0.2.

## Pistes v0.3

- **Parsing des documents non purement textuels** : diagrammes Mermaid,
   schémas d'architecture. Aujourd'hui ignorés → risque de faux positifs sur
   des infos présentes visuellement dans le doc.
- **Contrat du « draft figé » en cours de session** : garantir une maturité
   minimale avant appel, pour que les questions soient reproductibles et non
   du bruit sur un draft trop précoce.
