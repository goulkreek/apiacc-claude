# Conventions GraphQL

## Principes Généraux

### Localisation

**Fichiers** : `packages/backend/src/model/[domain]/[domain].graphql`

### Types GraphQL vs Mongoose

**Séparation stricte** :
- Types Mongoose : Préfixe `MG`, représentation DB
- Types GraphQL : Pas de préfixe, représentation API

### Transformation

Via fonctions `to[Entity]()` dans `[entity].transformer.ts`

## Nomenclature

### Types

**PascalCase** :
```graphql
type SiteAudit { }
type Customer { }
type MeasurementSession { }
```

### Champs

**camelCase** :
```graphql
type SiteAudit {
  id: ID!
  customerId: ID!
  createdAt: DateTime!
}
```

### Inputs

**Suffixe Input** :
```graphql
input CreateSiteAuditInput {
  name: String!
  customerId: ID!
}

input UpdateSiteAuditInput {
  name: String
  status: SiteAuditStatus
}

input SiteAuditFilterInput {
  customerId: ID
  status: SiteAuditStatus
}
```

### Enums

**UPPER_CASE pour valeurs** :
```graphql
enum SiteAuditStatus {
  SETUP
  MEASUREMENT_COLLECT
  MICROBIOLOGICAL_ANALYSIS
  READY_FOR_APPROVAL
  APPROVED
  REJECTED
  ARCHIVED
  CANCELLED
}

enum RoleEnum {
  ADMIN
  AGENCY_MANAGER
  AUDITOR
  APPROBATOR
  CUSTOMER
}
```

## Structure Type

### Pattern Standard

```graphql
type SiteAudit {
  # ID obligatoire (converti de ObjectId)
  id: ID!

  # Champs scalaires
  name: String!
  description: String
  status: SiteAuditStatus!

  # Dates (scalar DateTime)
  createdAt: DateTime!
  updatedAt: DateTime!

  # Relations (via DataLoader OBLIGATOIRE)
  customer: Customer!
  site: Site!
  devices: [Device!]!
  auditor: User
}
```

### Scalars Personnalisés

**DateTime** :
```graphql
scalar DateTime

# Représentation ISO 8601 string
# Exemple: "2025-01-24T10:30:00.000Z"
```

**JSON** (éviter si possible) :
```graphql
scalar JSON

# Utiliser uniquement pour données vraiment dynamiques
# Préférer des types stricts
```

## Queries

### Pattern Standard

```graphql
type Query {
  # Single item (nullable)
  siteAudit(id: ID!): SiteAudit

  # Liste (non-nullable array avec items non-nullable)
  siteAudits(filter: SiteAuditFilterInput): [SiteAudit!]!

  # Avec pagination
  siteAuditsPaginated(
    page: Int!
    limit: Int!
    filter: SiteAuditFilterInput
  ): SiteAuditPaginatedResponse!
}
```

### Response Paginée

```graphql
type SiteAuditPaginatedResponse {
  items: [SiteAudit!]!
  totalCount: Int!
  page: Int!
  limit: Int!
  hasMore: Boolean!
}
```

## Mutations

### Pattern Standard

```graphql
type Mutation {
  # Create - retourne l'objet créé
  createSiteAudit(input: CreateSiteAuditInput!): SiteAudit!

  # Update - retourne l'objet modifié
  updateSiteAudit(id: ID!, input: UpdateSiteAuditInput!): SiteAudit!

  # Delete - retourne true/false
  deleteSiteAudit(id: ID!): Boolean!

  # Actions métier - retourne l'objet modifié
  approveSiteAudit(id: ID!): SiteAudit!
  startMeasurementSession(auditId: ID!): MeasurementSession!
}
```

## Inputs

### Create Input

```graphql
input CreateSiteAuditInput {
  # Champs obligatoires
  name: String!
  customerId: ID!
  siteId: ID!

  # Champs optionnels
  description: String
  deviceIds: [ID!]

  # Jamais inclure id, createdAt, updatedAt (générés côté serveur)
}
```

### Update Input

```graphql
input UpdateSiteAuditInput {
  # Tous les champs optionnels (partial update)
  name: String
  description: String
  status: SiteAuditStatus
  deviceIds: [ID!]

  # Jamais inclure id, createdAt, updatedAt
}
```

### Filter Input

```graphql
input SiteAuditFilterInput {
  # Filtres de recherche
  customerId: ID
  siteId: ID
  status: SiteAuditStatus
  auditorId: ID

  # Filtres de date
  createdAfter: DateTime
  createdBefore: DateTime

  # Recherche textuelle
  searchTerm: String
}
```

## Relations

### One-to-One

```graphql
type SiteAudit {
  id: ID!
  # Relation one-to-one
  customer: Customer!  # Via customerId avec DataLoader
}
```

**Resolver** :
```typescript
@ResolveField(() => Customer)
async customer(@Parent() audit: SiteAudit): Promise<Customer> {
  return this.customerLoader.load(audit.customerId);
}
```

### One-to-Many

```graphql
type Customer {
  id: ID!
  name: String!

  # Relation one-to-many
  audits: [SiteAudit!]!  # Via query avec filter
}
```

**Resolver** :
```typescript
@ResolveField(() => [SiteAudit])
async audits(@Parent() customer: Customer): Promise<SiteAudit[]> {
  return this.siteAuditService.findByCustomerId(customer.id);
}
```

### Many-to-Many

```graphql
type SiteAudit {
  id: ID!

  # Relation many-to-many (IDs array)
  devices: [Device!]!  # Via deviceIds avec DataLoader
}
```

**Resolver** :
```typescript
@ResolveField(() => [Device])
async devices(@Parent() audit: SiteAudit): Promise<Device[]> {
  return this.deviceLoader.loadMany(audit.deviceIds);
}
```

## Nullability

### Règles

**Non-nullable (!)** :
- ID : `id: ID!`
- Champs obligatoires : `name: String!`
- Dates système : `createdAt: DateTime!`
- Relations obligatoires : `customer: Customer!`

**Nullable** :
- Champs optionnels : `description: String`
- Relations optionnelles : `approbator: User`
- Query single : `siteAudit(id: ID!): SiteAudit` (peut ne pas exister)

**Array non-nullable avec items non-nullable** :
```graphql
devices: [Device!]!  # Array jamais null, items jamais null
```

## Erreurs

### Union Types pour Erreurs

```graphql
type SiteAudit { }

type NotFoundError {
  message: String!
  code: String!
}

type UnauthorizedError {
  message: String!
  code: String!
}

union SiteAuditResult = SiteAudit | NotFoundError | UnauthorizedError

type Query {
  siteAudit(id: ID!): SiteAuditResult!
}
```

**Resolver** :
```typescript
@Query(() => SiteAuditResult)
async siteAudit(@Args('id') id: string): Promise<typeof SiteAuditResult> {
  const audit = await this.service.findById(id);

  if (!audit) {
    return {
      __typename: 'NotFoundError',
      message: 'Audit not found',
      code: 'NOT_FOUND',
    };
  }

  return audit;
}
```

## Conventions Spécifiques au Domaine

### Entités de Site

```graphql
enum SiteEntityTypeEnum {
  AREA
  PSM
  HFL
  ISOLATOR
  LAF
  GRILL
}

type SiteEntity {
  id: ID!
  type: SiteEntityTypeEnum!
  name: String!

  # Polymorphisme selon type
  areaSpecific: AreaSpecificData
  cabinetSpecific: CabinetSpecificData
}
```

### Mesures

```graphql
enum MeasurementTypeEnum {
  PARTICLE
  MICROBIOLOGICAL
  AIR_VELOCITY
  PRESSURE
  TEMPERATURE
  HUMIDITY
}

type Measurement {
  id: ID!
  type: MeasurementTypeEnum!
  value: Float!
  unit: String!
  timestamp: DateTime!

  # Conformité
  isConform: Boolean!
  threshold: Float
}
```

### Snapshots

```graphql
type MeasurementSession {
  id: ID!

  # Snapshot volumin eux (JSON)
  snapshot: JSON  # Version actuelle : v43
  snapshotVersion: Int!  # 43
}
```

**Note** : Préférer migration vers GCS pour snapshots lourds

## Documentation GraphQL

### Descriptions

```graphql
"""
Représente un audit de conformité d'un site.
Gère le cycle de vie complet de l'audit.
"""
type SiteAudit {
  """Identifiant unique de l'audit"""
  id: ID!

  """
  Nom de l'audit.
  Doit être unique par client et par année.
  """
  name: String!

  """
  Statut actuel de l'audit dans le workflow.
  Voir enum SiteAuditStatus pour les valeurs possibles.
  """
  status: SiteAuditStatus!
}

"""
Crée un nouvel audit de conformité.
Nécessite les permissions AUDITOR ou ADMIN.
"""
createSiteAudit(input: CreateSiteAuditInput!): SiteAudit!
```

## Deprecation

### Champs Obsolètes

```graphql
type SiteAudit {
  id: ID!

  # Nouveau champ
  externalReference: String

  # Ancien champ (déprécié)
  legacyCode: String @deprecated(reason: "Utiliser externalReference à la place")
}
```

## Directives

### @skip et @include

```graphql
query GetAudit($id: ID!, $includeDevices: Boolean!) {
  siteAudit(id: $id) {
    id
    name
    devices @include(if: $includeDevices) {
      id
      name
    }
  }
}
```

## Exemples Complets

### CRUD Complet

```graphql
# Type
type Device {
  id: ID!
  name: String!
  serialNumber: String!
  deviceType: DeviceTypeEnum!
  customer: Customer!
  sensors: [DeviceSensor!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

# Enum
enum DeviceTypeEnum {
  PARTICLE_COUNTER
  AIR_SAMPLER
  THERMO_HYGROMETER
  ANEMOMETER
  MANOMETER
}

# Inputs
input CreateDeviceInput {
  name: String!
  serialNumber: String!
  deviceType: DeviceTypeEnum!
  customerId: ID!
}

input UpdateDeviceInput {
  name: String
  serialNumber: String
}

input DeviceFilterInput {
  customerId: ID
  deviceType: DeviceTypeEnum
  searchTerm: String
}

# Queries
type Query {
  device(id: ID!): Device
  devices(filter: DeviceFilterInput): [Device!]!
}

# Mutations
type Mutation {
  createDevice(input: CreateDeviceInput!): Device!
  updateDevice(id: ID!, input: UpdateDeviceInput!): Device!
  deleteDevice(id: ID!): Boolean!
}
```

## Résumé des Conventions

1. ✅ **Types** : PascalCase
2. ✅ **Champs** : camelCase
3. ✅ **Inputs** : Suffixe `Input`
4. ✅ **Enums** : Valeurs UPPER_CASE
5. ✅ **Relations** : Toujours via DataLoader
6. ✅ **Arrays** : `[Item!]!` (non-nullable)
7. ✅ **Queries** : Liste retourne `[T!]!`
8. ✅ **Mutations** : Create/Update retournent l'objet
9. ✅ **Dates** : Scalar `DateTime` (ISO 8601)
10. ✅ **Documentation** : Descriptions pour types et champs publics
