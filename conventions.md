# Conventions de développement pour Claude Code

## Format des messages de commit

### Structure obligatoire

```
type(scope): sujet du commit

- Description détaillée ligne 1
- Description détaillée ligne 2
- Description détaillée ligne 3

#<clickup-ticket-id>
```

### Règles strictes pour Claude Code

1. **ID du ticket ClickUp** :
   - **OBLIGATOIRE** : Toujours placé à la fin du corps du message
   - **INTERDIT** : Ne jamais le mettre dans le sujet du commit
   - Format : `#<ticket-id>` (exemple: `#86c5u1gpn`)
   - Extraire l'ID depuis le nom de la branche si elle suit le format `task/#<ticket-id>-description`
   - Si pas d'ID de ticket déterminé, ne pas commiter et indiquer cela à l'utilisateur

2. **Type de commit** : `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, etc.

3. **Scope** :
   - `bo` pour backoffice
   - `audit` pour audit
   - `measurements` pour measurements
   - `backend` pour backend
   - `docgen` pour docgen
   - Autres scopes selon le contexte

4. **Attribution** :
   - Ne JAMAIS ajouter la mention "Generated with Claude Code"
   - Ne JAMAIS ajouter "Co-Authored-By: Claude"

### Exemples de commits

#### ✅ CORRECT

```
fix(bo): correct shape.type becoming null on second render

- Fix validation schema to use 'shapeType' instead of 'type'
- Update conditional validations in form-item-shape-schema
- Ensure consistency between schema definition and usage

#86c5u1gpn
```

```
test(bo): add comprehensive unit tests for form-item-shape schema

- Add tests for all three shape types (triangle, rectangle, circle)
- Test validation of required fields for each shape type
- Test rejection of invalid, null, and undefined shapeType values

#86c5u1gpn
```

#### ❌ INCORRECT

```
fix(bo): correct shape.type becoming null (#86c5u1gpn)

- Fix validation schema
```

```
fix(bo): correct shape.type becoming null

- Fix validation schema

#86c5u1gpn
```

## Workflow de commit

Quand l'utilisateur demande de créer un commit :

1. Vérifier le nom de la branche actuelle avec `git branch --show-current`
2. Extraire l'ID du ticket si le format est `task/#<ticket-id>-*`
3. Créer le message de commit avec l'ID à la fin du corps
4. Ne jamais ajouter d'attribution Claude Code
