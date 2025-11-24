# Configuration Claude Code - APIACC

Répertoire de configuration pour [Claude Code](https://claude.com/claude-code) du projet **APIACC** (Application de Pilotage et d'Inventaire des Appareils de Chauffage et de Combustion).

## Vue d'ensemble

Ce répertoire contient toute la configuration nécessaire pour que Claude Code puisse travailler efficacement sur le projet APIACC, incluant :

- **Système multi-agents** : 8 agents spécialisés pour l'orchestration automatique des tickets ClickUp
- **Base de connaissances** : Documentation de l'architecture, conventions, et patterns du projet
- **Conventions** : Règles de développement et standards de code
- **Configuration** : Settings personnalisés pour Claude Code

## Structure du Répertoire

```
.claude/
├── README.md                     # Ce fichier
├── settings.local.json           # Configuration Claude Code
├── conventions.md                # Conventions de développement (commits, workflow)
├── MCP_PLAYWRIGHT_GUIDE.md      # Guide d'utilisation de Playwright MCP
│
├── agents/                       # Système multi-agents v2.0 (8 agents)
│   ├── README.md                # Documentation complète du système
│   ├── coordinateur.md          # Orchestrateur suprême
│   ├── analyste-ticket.md       # Analyse des tickets ClickUp
│   ├── architecte.md            # Validation architecture
│   ├── metier.md                # Expert métier (audit + measurements)
│   ├── database-expert.md       # Migrations MongoDB
│   ├── developpeur.md           # Développement full-stack
│   ├── qa.md                    # Tests fonctionnels et visuels
│   ├── simulation.md            # Tests en situation réelle
│   └── memory.md                # Gestion de la mémoire du système
│
└── memory/                       # Base de connaissances du projet
    ├── architecture.md          # Architecture générale du monorepo
    ├── conventions_typescript.md # Conventions TypeScript strictes
    ├── backend_patterns.md      # Patterns backend (NestJS/GraphQL/MongoDB)
    ├── frontend_patterns.md     # Patterns frontend (React/MobX/Ant Design)
    ├── graphql_schemas.md       # Schémas GraphQL
    ├── mongoose_models.md       # Modèles Mongoose
    ├── domain_audit.md          # Domaine métier : audits et conformité
    ├── domain_measurements.md   # Domaine métier : mesures et calculs
    ├── code_generation_rules.md # Règles de génération de code
    └── ui_ux_rules.md           # Règles UI/UX et accessibilité
```

## Système Multi-Agents v2.0

### Présentation

Le répertoire `agents/` contient un système optimisé de **8 agents intelligents** capables de traiter automatiquement les tickets ClickUp du début à la fin :

1. **coordinateur** : Orchestrateur suprême qui gère le workflow complet
2. **analyste-ticket** : Analyse et décompose les tickets en plan d'action
3. **architecte** : Valide les décisions d'architecture et patterns
4. **metier** : Expert métier unifié (audit + measurements + conformité)
5. **database-expert** : Gère les migrations MongoDB
6. **developpeur** : Développeur full-stack unifié (backend + frontend + docgen)
7. **qa** : Tests fonctionnels et visuels unifiés
8. **simulation** : Tests en situation réelle

### Utilisation

Pour traiter un ticket ClickUp automatiquement :

```bash
# Via ID de ticket
Traite le ticket #86c6pc26v

# Via URL ClickUp
Traite le ticket https://app.clickup.com/t/86c6pc26v
```

Le coordinateur prend le contrôle et orchestre tous les agents nécessaires pour :
- Analyser le ticket
- Implémenter la fonctionnalité ou corriger le bug
- Exécuter les tests (unitaires, fonctionnels, visuels, réels)
- Créer la pull request

Pour plus de détails, consulter [`agents/README.md`](agents/README.md).

## Base de Connaissances (Memory)

Le répertoire `memory/` contient la documentation technique du projet APIACC :

### Architecture
- **architecture.md** : Structure du monorepo, packages, technologies
- **backend_patterns.md** : Patterns NestJS, GraphQL, MongoDB
- **frontend_patterns.md** : Patterns React, MobX, Ant Design
- **graphql_schemas.md** : Schémas et types GraphQL
- **mongoose_models.md** : Modèles de données MongoDB

### Métier
- **domain_audit.md** : Logique métier des audits de conformité
- **domain_measurements.md** : Logique métier des mesures et calculs

### Conventions et Règles
- **conventions_typescript.md** : Standards TypeScript stricts (pas de cast)
- **code_generation_rules.md** : Règles pour la génération de code
- **ui_ux_rules.md** : Standards UI/UX et accessibilité

## Conventions de Développement

### Format des Commits

Le fichier [`conventions.md`](conventions.md) définit le format strict des commits :

```
type(scope): sujet du commit

- Description détaillée ligne 1
- Description détaillée ligne 2
- Description détaillée ligne 3

#<clickup-ticket-id>
```

**Règles importantes** :
- L'ID du ticket ClickUp est **OBLIGATOIRE** à la fin du corps
- **INTERDIT** de mettre l'ID dans le sujet du commit
- **INTERDIT** d'ajouter les mentions "Generated with Claude Code" ou "Co-Authored-By: Claude"
- Types : `feat`, `fix`, `refactor`, `test`, `docs`, `chore`
- Scopes : `bo`, `audit`, `measurements`, `backend`, `docgen`

### Exemple de Commit Correct

```
fix(bo): correct shape.type becoming null on second render

- Fix validation schema to use 'shapeType' instead of 'type'
- Update conditional validations in form-item-shape-schema
- Ensure consistency between schema definition and usage

#86c5u1gpn
```

## Configuration Claude Code

Le fichier `settings.local.json` contient la configuration personnalisée de Claude Code pour ce projet, incluant :

- Paramètres des MCP (Model Context Protocol) servers
- Hooks personnalisés
- Préférences de l'utilisateur

## Guides Spécifiques

### MCP Playwright
Le fichier [`MCP_PLAYWRIGHT_GUIDE.md`](MCP_PLAYWRIGHT_GUIDE.md) contient le guide complet d'utilisation du serveur MCP Playwright pour l'automatisation des tests browser.

## Technologies Projet APIACC

### Backend
- **NestJS** : Framework backend Node.js
- **GraphQL** : API avec Apollo Server
- **MongoDB** : Base de données avec Mongoose
- **TypeScript** : Langage strict (pas de cast autorisé)

### Frontend
- **React** : Bibliothèque UI
- **MobX** : State management
- **Ant Design** : Composants UI
- **Next.js** : Framework pour l'application Eole

### Outils
- **Yarn** : Gestionnaire de packages
- **Turborepo** : Build system pour monorepo
- **Jest** : Tests unitaires
- **Playwright** : Tests end-to-end

## Règles Globales

### Qualité du Code
- ⛔ **Aucun cast TypeScript** autorisé (`as`, `<Type>`)
- ✅ Respect strict des conventions du projet
- ✅ Tests unitaires obligatoires pour tout nouveau code
- ✅ Code compilé sans erreur

### CI/CD
- ⛔ Aucune modification de `.gitlab-ci.yml`
- ⛔ Aucune modification des scripts de déploiement
- ⛔ Pas de déploiement direct

### Documentation
- Documentation créée **uniquement sur demande explicite**
- Pas de création proactive de README ou ADR

## Maintenance

### Ajouter de la Connaissance

Pour ajouter de la documentation au projet :

```bash
# Ajouter un fichier dans memory/
echo "# Nouveau domaine" > memory/nouveau_domaine.md
```

### Modifier un Agent

Pour modifier le comportement d'un agent :

```bash
# Éditer le fichier de l'agent
vi agents/nom-agent.md

# Mettre à jour le coordinateur si nécessaire
vi agents/coordinateur.md
```

### Mettre à Jour les Conventions

Pour modifier les conventions de développement :

```bash
vi conventions.md
```

## Support et Ressources

- **Documentation Claude Code** : [https://docs.anthropic.com/claude/docs/claude-code](https://docs.anthropic.com/claude/docs/claude-code)
- **Issues Claude Code** : [https://github.com/anthropics/claude-code/issues](https://github.com/anthropics/claude-code/issues)
- **Système Multi-Agents** : Voir [`agents/README.md`](agents/README.md)

## Changelog

### Version 2.0 (2025-01-24)
- Optimisation du système multi-agents : 13 → 8 agents (-38%)
- Agents unifiés : developpeur, metier, qa
- Nouvel agent simulation pour tests réels
- Workflow simplifié

### Version 1.0 (2025-01-24)
- Création initiale du système multi-agents
- 13 agents spécialisés
- Base de connaissances complète

---

**Configuration opérationnelle pour Claude Code sur le projet APIACC**
