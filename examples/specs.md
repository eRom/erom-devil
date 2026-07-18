# Specs techniques — veilleur v1

## Architecture

- CLI TypeScript (bun) qui dialogue avec un backend SaaS multi-tenant hébergé
  sur Fly.io.
- Les comptes utilisateurs (email + mot de passe) sont gérés par le backend ;
  chaque machine synchronise ses flux via l'API du backend.
- Le stockage local utilise des fichiers JSON plats dans `~/.veilleur/`.

## Résumés

- Les résumés sont générés par l'API OpenAI (gpt-4o-mini).
- La clé API OpenAI est stockée en clair dans `config.toml`.

## Modèle de données

- Les articles sont persistés dans une base SQLite locale.
- Schéma : `feeds(id, url, tags)`, `articles(id, feed_id, title, url,
  published_at, summary)`.

## Config

- Fichier TOML : liste des flux, cadence de fetch, mots-clés de filtrage.

## Notifications

- Le système notifie l'utilisateur au bon moment.

## Commandes

- `veilleur digest` : produit le résumé du jour.
- `veilleur add <url>` : ajoute un flux.
