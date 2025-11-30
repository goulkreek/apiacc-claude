# Mémoire Projet APIACC

## Règles générales

- Écrire toujours en français
- Ne jamais utiliser `any` en TypeScript

## Règles Git

### Messages de commit
- **Ne JAMAIS mentionner Claude, Claude Code, ou toute IA** dans les messages de commit
- Pas de "Generated with Claude Code", "Co-Authored-By: Claude", ou similaire
- Les commits doivent paraître écrits par un développeur humain
- Format : `type(scope): description concise`
  - Types : feat, fix, refactor, docs, test, chore
  - Exemple : `feat(auth): ajouter la connexion OAuth2`

## Règles React

### Formulaires
- **Toujours utiliser react-hook-form avec zod** comme resolver
- Ne jamais utiliser yup, class-validator ou validation manuelle côté frontend

### Architecture des composants
- Maximiser la réutilisabilité : découper en petits composants atomiques
- Pattern de composition : UI (visuels) → Container (logique) → Feature (assemblage)
- Extraire les composants dès qu'ils sont utilisés plus d'une fois
- Single Responsibility Principle

### Éviter la duplication
- Vérifier si un composant similaire existe avant d'en créer un nouveau
- Si existant couvre 70%+ du besoin, l'adapter plutôt que créer
- Techniques : props `variant`, `allowedXXX`, `renderXXX`, composition avec `children`
- Rétrocompatibilité : nouvelles props avec valeurs par défaut

## Backend - Migrations MongoDB

### Créer une nouvelle migration

1. Créer le fichier TypeScript dans `packages/backend/db-migrations/ts/` :
   ```
   YYYYMMDDHHMMSS-description.ts
   ```
   Exemple : `20251129150927-populate-source-site-audit-id-from-series.ts`

2. Structure du fichier :
   ```typescript
   import type { Db, MongoClient } from 'mongodb'

   module.exports = {
     async up(db: Db, client: MongoClient) {
       // Migration code
     },
     async down(db: Db, client: MongoClient) {
       // Rollback code
     },
   }
   ```

3. Builder les migrations :
   ```bash
   yarn build:dbmigrations
   ```
   Cela compile tous les `.ts` de `db-migrations/ts/` vers `db-migrations/` en JS.

4. Exécuter :
   ```bash
   yarn migrate up      # Appliquer
   yarn migrate down    # Rollback
   ```

### Bonnes pratiques migrations
- Toujours implémenter `down()` pour le rollback
- Utiliser `console.time()` / `console.timeLog()` pour le suivi de progression
- Valider les données avant modification (countDocuments avant updateMany)
- Lever une erreur si données corrompues détectées
