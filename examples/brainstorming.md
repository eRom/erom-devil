# Brainstorming — veilleur, CLI de veille RSS locale

## Intention

Un outil CLI personnel qui agrège mes flux RSS techniques et produit un résumé
quotidien lisible en 2 minutes. Local d'abord : AUCUNE donnée envoyée dans le
cloud, c'est non négociable (les flux lus dessinent un profil intellectuel).

## Idées retenues

- CLI TypeScript exécutée via bun, installable en binaire unique.
- Config déclarative en un fichier TOML : liste des flux, cadence, mots-clés.
- Résumés générés par un LLM LOCAL (ollama), jamais par une API distante.
- Notification macOS native quand le digest du jour est prêt.
- Export hebdomadaire en markdown dans un dossier choisi (archive perso).
- Stockage minimal : fichiers plats, pas de serveur, pas de daemon.

## Non-objectifs

- Pas de multi-utilisateurs, pas de comptes, pas de sync entre machines.
- Pas d'interface web.

## Critères de succès

- `veilleur digest` produit le résumé du jour en moins de 30 s sur un M1.
- Zéro requête réseau sortante hors fetch des flux RSS eux-mêmes.
- L'export hebdo est du markdown propre, lisible dans Obsidian.
