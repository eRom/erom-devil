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
