Grille de standards LIANTE pour ce repo (stack : Next 16, Prisma 7,
Postgres, Zod 4, next-safe-action, Vitest, Playwright, Biome). Chaque
violation est une issue normale : ancrée file:ligne, failure_scenario
concret, sévérité et catégorie indiquées par la règle. Ne flagge que ce que
le DIFF introduit ou touche.

## Autorisation (violation : severity critical, category security)

- L'autorisation ne vit JAMAIS dans `middleware.ts` ni `proxy.ts`
  (CVE-2025-29927 : un header forgé contournait entièrement le middleware).
  Tout accès aux données passe par la DAL (`src/lib/dal.ts`).
- Trois checks distincts par mutation : session, permission, propriété.
  L'absence du check de PROPRIÉTÉ est LE défaut à chercher : c'est lui qui
  produit les IDOR.
- La propriété vit dans la clause `where` de la requête, jamais dans un
  `if` préalable (supprime aussi la course entre vérification et écriture).
- Toute Server Action est un endpoint POST public : aucune sans
  `actionClient` (next-safe-action, `src/lib/safe-action.ts`).
- Un fichier ne transite jamais par une Server Action : URL présignée,
  upload direct du client vers le stockage objet.

## Pièges de version (violation : severity high, category correctness)

- Prisma 7 : `url = env("DATABASE_URL")` dans le bloc datasource est un
  refus de validation (P1012). La connexion Migrate vit dans
  `prisma.config.ts`, `PrismaClient` reçoit un adapter (`@prisma/adapter-pg`).
  Aucune API du moteur Rust (supprimé en v7), aucune API Prisma 8 (RC).
- Zod 4 : `z.email()` et `z.url()` au top-level. `z.string().email()` et
  `z.string().url()` sont dépréciés : ils valident encore en 4.x, la
  suppression arrive en v5. Leur présence dans le diff est une issue.
- Next 16 : types de route globaux (`LayoutProps<"/">`, `PageProps<...>`).
  Pages Router, `getServerSideProps`, `getStaticProps` et `next/head` sont
  interdits.
- Node est requis au build ET à l'exécution ; Bun ne sert qu'à installer.
  `next start` ne fonctionne pas avec `output: "standalone"`.

## Architecture (violation : severity medium, category architecture)

- La logique métier vit dans la DAL et des fonctions pures, pas dans les
  composants.
- Un seul schéma Zod par formulaire, partagé client et serveur.
- `next/image`, jamais `<img>`.
- Aucune API Vercel-only (`@vercel/kv`, `@vercel/blob`, `@vercel/postgres`) :
  `output: "standalone"` et le Dockerfile doivent rester fonctionnels.
- Toute variable d'environnement passe par `src/env.ts` ; logs en JSON
  structuré, jamais de PII ni de secret dedans.

## Tests (violation : severity medium à high, category tests)

- Un Server Component async ne se rend pas sous Vitest (limite documentée
  Next) : un test Vitest qui le tente est une issue ; ces composants se
  testent sous Playwright.
- Antipatterns interdits : test change-detector (compteur ou snapshot de
  catalogue figé) et test qui asserte sur le contenu du code source. On
  assert sur le comportement.
- Le seed est déterministe : aucune valeur aléatoire non seedée dans une
  fixture.
