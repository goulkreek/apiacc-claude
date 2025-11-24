# Architecture Globale du Projet APIACC

## Vue d'Ensemble

APIACC est un monorepo Lerna + Yarn Workspaces pour la gestion d'audits de conformité en salles propres et zones contrôlées.

### Secteur

- Pharmaceutique, hospitalier, microélectronique
- Contrôle de contamination particulaire et microbiologique
- Conformité aux normes ISO 14644, BPF, COFRAC

## Structure du Monorepo

```
apiacc-monorepo-second/
├── packages/
│   ├── backend/           # API NestJS + GraphQL + MongoDB
│   ├── backoffice/        # Application React (backoffice admin)
│   ├── audit/             # Logique métier conformité (pure TS)
│   ├── measurements/      # Logique métier mesures (pure TS)
│   ├── docgen/            # Génération rapports PDF
│   └── utils/             # Outils CLI
├── eole/                  # Portail client Next.js (hors packages/)
├── .claude/
│   ├── agents/            # Agents spécialisés
│   └── memory/            # Mémoire projet
└── lerna.json
```

## Packages et Responsabilités

### Backend (`packages/backend/`)

**Rôle** : API GraphQL, logique serveur, persistance MongoDB

**Technologies** :
- NestJS 11.1.9
- GraphQL (Apollo Server)
- Mongoose 6.13.8
- TypeScript 5.9.3

**Structure** :
```
packages/backend/src/
├── model/                 # Modules NestJS par domaine
│   ├── site-audit/        # Schéma, Service, Resolver, Module
│   ├── customer/
│   ├── device/
│   └── ...
├── common/                # Utilitaires partagés
├── auth/                  # Authentification (Passport)
└── db-migrations/         # Scripts migration MongoDB
```

**Dépendances** : `audit`, `measurements`, `docgen`

### Backoffice (`packages/backoffice/`)

**Rôle** : Interface d'administration web

**Technologies** :
- React 18.3.1
- MobX State Tree 7.0.2
- Ant Design 5.29.1
- Apollo Client 3.7.17

**Structure** :
```
packages/backoffice/src/
├── pages/                 # Pages de routage
├── containers/            # Composants avec logique
├── views/                 # Composants réutilisables
├── components/            # Composants UI purs
├── store/                 # MobX stores
└── services/              # Services (API, GraphQL)
```

**Dépendances** : `audit`, `measurements` (types uniquement)

### Eole (`eole/`)

**Rôle** : Portail client pour consultation de rapports

**Technologies** :
- Next.js 13.5.11
- React 18.3.1
- Visualisation données (visx/D3)

**Déploiement** : Vercel

### Audit (`packages/audit/`)

**Rôle** : Logique métier conformité (pure TypeScript)

**Responsabilités** :
- Règles de conformité (ISO, BPF, COFRAC)
- Protocoles de tests
- Validation des habilitations
- Calculs de seuils normatifs

**Principe** : Aucune dépendance UI ou DB, uniquement types et fonctions pures

### Measurements (`packages/measurements/`)

**Rôle** : Logique métier mesures (pure TypeScript)

**Responsabilités** :
- Types de mesures (particules, micro, aéraulique)
- Calculs de conformité
- Transformateurs de formats (v1 → v43)
- Génération conclusions automatiques

**Complexité** : 43 versions de formats accumulées avec migrations bidirectionnelles

### Docgen (`packages/docgen/`)

**Rôle** : Génération de rapports PDF

**Technologies** :
- @react-pdf/renderer 3.4.5
- Recharts (graphiques)
- Sharp (images)

**Dépendances** : `audit`, `measurements`

## Flux de Données Principaux

### 1. Création d'Audit

```
Backoffice (React)
  → Mutation GraphQL (Apollo Client)
  → Backend (NestJS Resolver)
  → Service (validation métier via audit package)
  → MongoDB (Mongoose)
  → Retour GraphQL
  → Mise à jour UI (MobX)
```

### 2. Saisie de Mesures

```
Backoffice (Interface saisie)
  → Calcul conformité (measurements package)
  → Snapshot JSON (sauvegarde état)
  → Mutation GraphQL
  → Backend (sauvegarde snapshot dans MeasurementSession)
  → MongoDB (document avec snapshot volumineux)
```

### 3. Génération de Rapport PDF

```
Backoffice (Bouton "Approuver")
  → Mutation GraphQL (avec données complètes)
  → Backend (génération via docgen package)
  → Templates PDF (@react-pdf/renderer)
  → Intégration graphiques (Recharts → Canvas → Image)
  → Upload vers GCS (Google Cloud Storage)
  → Sauvegarde URL en DB
  → Retour URL signée
  → Download depuis Backoffice/Eole
```

**Durée** : 2-5 minutes selon complexité

### 4. Consultation Client (Eole)

```
Eole (Next.js)
  → Authentification (JWT)
  → Requête GraphQL (rapports client)
  → Backend (filtrage par customerId)
  → Retour liste rapports
  → Affichage + Download PDF
```

## Patterns de Communication

### Backend ↔ Frontend

**Protocole** : GraphQL (mutations + queries)

**Optimisation** : DataLoaders obligatoires pour toutes les relations (éviter N+1)

**État** : MobX State Tree côté frontend

### Backend ↔ Packages Métier

**Import direct** :
```typescript
// Dans backend service
import { checkParticleConformity } from '@igienairco/apiacc-audit';
import { transformV42ToV43 } from '@igienairco/apiacc-measurements';
```

**Principe** : Packages métier sont des bibliothèques de fonctions pures

### Frontend ↔ Packages Métier

**Import direct** :
```typescript
// Dans composant React
import { calculateStatistics } from '@igienairco/apiacc-measurements';
```

**Usage** : Calculs côté client pour réactivité (affichage temps réel)

## Déploiement

### Production

**Backend** : Google Cloud Run
**Backoffice** : Google Cloud Run
**Eole** : Vercel
**MongoDB** : MongoDB Atlas
**Storage** : Google Cloud Storage (rapports PDF)

### CI/CD

**Outil** : GitLab CI
**Fichier** : `.gitlab-ci.yml` (NE JAMAIS MODIFIER par agents)

**Pipelines** :
- Build & Test
- Deploy Staging
- Deploy Production

## Entités Métier Principales

### SiteAudit (Cœur du système)

**États** : 8 étapes du cycle de vie
```
setup → measurementCollect → microbiologicalAnalysis →
readyForApproval → approved → rejected → archived → cancelled
```

**Relations** :
- Customer (client)
- Site (site audité)
- SiteEntities (zones/équipements)
- Devices (dispositifs de mesure)
- Users (auditeur, approbateur)
- MeasurementSession (données de mesures)
- SiteAuditReport (rapport PDF)

### Types d'Entités de Site

- **Areas** : Salles, couloirs, SAS
- **Cabinets** : PSM (Postes de Sécurité Microbiologique), HFL (Hotte à Flux Laminaire), Isolateurs
- **Grills** : Grilles d'extraction/soufflage

### Types de Mesures

1. **Particules** : Comptage par taille (0.3 µm, 0.5 µm, 1 µm, 5 µm)
2. **Microbiologie** : UFC/m³ (air), UFC/gélose (surfaces)
3. **Aéraulique** : Vitesse air, pression différentielle, taux de brassage
4. **Tests spécialisés** : Turbulences, confinement, intégrité filtres

## Principes Architecturaux

### 1. Séparation Backend/Frontend

- Backend : API GraphQL pure, pas de logique UI
- Frontend : Consommation GraphQL, pas de logique métier complexe
- Métier : Packages séparés, réutilisables

### 2. Logique Métier Pure

- Packages `audit` et `measurements` : TypeScript pur
- Pas de dépendances UI (React) ou DB (Mongoose)
- Testabilité maximale (> 90% couverture)

### 3. DataLoaders Obligatoires

- Toute relation GraphQL doit utiliser un DataLoader
- Éviter le problème N+1
- Performance critique (requêtes multiples)

### 4. Typage Strict

- Aucun cast TypeScript autorisé (`as`, `<Type>`)
- Conventions strictes (préfixe MG pour Mongoose, etc.)

### 5. Ant Design Exclusif

- Frontend : Utiliser uniquement composants Ant Design
- Pas de custom CSS sauf Tailwind pour utilitaires
- Cohérence visuelle garantie

## Conventions de Nommage

### Backend

- Schémas Mongoose : Préfixe `MG` (ex: `MGSiteAudit`)
- Collections MongoDB : lowercase, plural (ex: `siteaudits`)
- Services : Suffixe `Service` (ex: `SiteAuditService`)
- Resolvers : Suffixe `Resolver` (ex: `SiteAuditResolver`)

### Frontend

- Composants : PascalCase (ex: `SiteAuditList`)
- Stores MobX : Suffixe `Store` (ex: `SiteAuditStore`)
- Hooks : Préfixe `use` (ex: `useStores`)

### GraphQL

- Types : PascalCase (ex: `SiteAudit`)
- Champs : camelCase (ex: `createdAt`)
- Inputs : Suffixe `Input` (ex: `CreateSiteAuditInput`)
- Queries/Mutations : camelCase (ex: `createSiteAudit`)

## Dépendances Inter-Packages

```
backend → audit, measurements, docgen
backoffice → audit, measurements (types uniquement)
docgen → audit, measurements
eole → (via API GraphQL)
```

**Principe** : Dépendances unidirectionnelles, pas de dépendances circulaires

## Points d'Attention

### Performance

- Génération PDF : 2-5 min (acceptable mais à optimiser si possible)
- Snapshots volumineux : 1-5 MB par session (migration vers GCS recommandée)
- DataLoaders : Critiques pour performance GraphQL

### Complexité Métier

- 43 versions de formats de données (legacy lourd)
- Normes multiples : ISO 14644, BPF, COFRAC
- Calculs scientifiques précis requis

### Réglementaire

- Traçabilité obligatoire (COFRAC)
- Incertitudes de mesure à documenter
- Conformité stricte aux normes

## Évolutions Futures Envisagées

- Migration snapshots MongoDB → GCS
- Optimisation génération PDF (< 1 min)
- Tests automatisés end-to-end (Playwright)
- Amélioration mode hors-ligne
