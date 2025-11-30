# Outil de Vérification d'Intégrité des Snapshots

## Concept de Série d'Audit (siteAuditSeriesId)

Les audits peuvent être liés entre eux via le champ `source.siteAuditSeriesId` dans SiteAudit :

```typescript
type SiteAuditSource = {
  siteAuditSeriesId: ID!  // Identifiant de la série
  siteAuditId: ID!        // Audit source original
}
```

- **Audits liés** : Plusieurs audits peuvent partager le même `siteAuditSeriesId`
- **Snapshots** : Chaque audit peut avoir plusieurs snapshots de session de mesures
- **Cohérence inter-snapshots** : Les IDs de siteEntity et openings doivent être cohérents au sein d'une série

### Structure des Snapshots

```typescript
type MeasurementSessionSnapshotSummary = {
  id: ID!
  siteAuditId: ID!
  siteAuditSeriesId: ID      // Peut être null pour audits non liés
  siteAuditName: String!
  dataVersion: Int!
  createdAt: Date!
}
```

## Emplacement

`packages/backoffice/src/services/integrity-check/`

## Types de vérifications (IntegrityCheckType)

1. **duplicateOpeningIds** : Détecte les IDs d'ouvertures dupliqués dans un snapshot
2. **modelValidation** : Valide la structure du modèle de données (champs requis, types)
3. **mstInstantiation** : Vérifie que les données peuvent instancier le modèle MST
4. **crossSnapshotConsistency** : Vérifie la cohérence des IDs entre snapshots d'une série

## Types d'issues (IntegrityCheckIssueType)

- `duplicate_opening_id` : ID d'ouverture en double
- `model_validation_error` : Erreur de validation du modèle
- `transformation_error` : Erreur de parsing/transformation JSON
- `mst_instantiation_error` : Échec d'instanciation MST
- `cross_snapshot_inconsistency` : Incohérence entre snapshots d'une série

### Snapshots impactés (impactedSnapshots)

Chaque issue peut inclure une liste des autres snapshots de la série impactés :

```typescript
type ImpactedSnapshotRef = {
  snapshotId: string
  siteAuditId: string
  siteAuditName: string
  createdAt: Date
}

type IntegrityCheckIssue = {
  type: IntegrityCheckIssueType
  severity: 'error' | 'warning'
  message: string
  details?: Record<string, unknown>
  impactedSnapshots?: ImpactedSnapshotRef[]  // Autres snapshots à corriger ensemble
}
```

Cette information est automatiquement ajoutée par `checkAuditSeriesIntegrity` pour les issues liées à un siteEntity (ex: `duplicate_opening_id`). Elle permet de savoir quels snapshots devront être corrigés de manière cohérente.

## Fichiers principaux

```
services/integrity-check/
  types.ts                           # Types et interfaces
  check-snapshot-integrity.ts        # Vérification individuelle d'un snapshot
  check-audit-series-integrity.ts    # Vérification par série d'audit
  fix-duplicate-opening-ids.ts       # Correction automatique des doublons
  index.ts                           # Exports
```

## Fonctions clés

### Vérification

```typescript
// Vérifier l'intégrité d'un snapshot
checkSnapshotIntegrity(snapshot, options?) → SnapshotIntegrityCheckResult

// Grouper les snapshots par série
groupSnapshotsBySeries(snapshots) → Map<seriesId | null, AuditSeries>

// Vérifier la cohérence inter-snapshots dans une série
checkCrossSnapshotConsistency(snapshotsData) → IntegrityCheckIssue[]

// Vérifier une série d'audit complète
checkAuditSeriesIntegrity(seriesId, snapshots, options?) → AuditSeriesIntegrityCheckResult

// Extraire les IDs d'ouvertures d'un snapshot
extractOpeningIdsFromSnapshot(data) → Map<siteEntityId, Set<openingId>>
```

### Correction

```typescript
// Corriger les doublons d'opening IDs dans un snapshot unique
fixDuplicateOpeningIds(snapshotDataJson: string) → FixDuplicateOpeningIdsResult

// Corriger les opening IDs de manière cohérente dans toute une série
// Utilise les IDs du snapshot le plus ancien comme référence
fixDuplicateOpeningIdsInSeries(
  seriesId: string | null,
  snapshots: SeriesSnapshotInput[]
) → FixDuplicateOpeningIdsInSeriesResult
```

### Stratégie de correction par série

La fonction `fixDuplicateOpeningIdsInSeries` assure la cohérence des IDs d'ouvertures au sein d'une série d'audits :

1. **Référence** : Le snapshot le plus ancien (par `createdAt`) sert de référence
2. **Propagation** : Les IDs de référence sont appliqués à tous les snapshots plus récents
3. **Conservation** : Si un ID est déjà correct, aucun changement n'est effectué
4. **Résultat détaillé** : Chaque snapshot retourne la liste des changements appliqués

```typescript
type SeriesSnapshotInput = {
  snapshotId: string
  siteAuditId: string
  siteAuditName: string
  createdAt: Date
  data: string  // JSON
}

type FixDuplicateOpeningIdsInSeriesResult = {
  success: boolean
  seriesId: string | null
  totalSnapshots: number
  snapshotsFixed: number
  referenceOpeningIds: Map<string, Map<number, string>>  // siteEntityId → index → refId
  results: SeriesSnapshotFixResult[]
}
```

## Store MobX (diagnosticsStore.integrityCheck)

```typescript
// Propriétés
lastSummary: IntegrityCheckSummaryModel | null
pendingResults: SnapshotIntegrityCheckResultModel[]
enabledCheckTypes: string[]  // Types de vérifications activées

// Actions
setSummary(summary)
addPendingResult(result)
clearSummary()
clearPendingResults()
setEnabledCheckTypes(checkTypes)
toggleCheckType(checkType, enabled)
```

## Composants UI

- `IntegrityCheckTypeSelector` : Sélection des types de vérifications
- `MeasurementsSessionSnapshotIntegrityCheckCard` : Analyse batch de tous les snapshots
- `SnapshotIntegrityCheckModal` : Vérification individuelle d'un snapshot

## GraphQL - Requêtes liées

```graphql
# Récupérer les résumés de snapshots (inclut siteAuditSeriesId)
query measurementSessionSnapshotSummaries {
  measurementSessionSnapshotSummaries {
    id
    siteAuditId
    siteAuditSeriesId
    siteAuditName
    dataVersion
    createdAt
  }
}

# Récupérer un snapshot complet
query siteAuditMeasurementSessionSnapshot($id: ID!) {
  siteAuditMeasurementSessionSnapshot(id: $id) {
    id
    siteAuditId
    data
    createdAt
  }
}
```

## Commandes de développement

```bash
# Tests d'intégrité
yarn workspace apiacc-backoffice test "integrity-check"
```
