# Domaine Métier : Mesures

## Vue d'Ensemble

**Package** : `packages/measurements/`

**Rôle** : Logique métier pour les mesures environnementales

**Principe** : TypeScript pur, pas de dépendances UI ou DB

## Complexité Principale

### 43 Versions de Formats

**Historique** : Accumulation de versions depuis le début du projet

**Problématique** :
- Compatibilité ascendante obligatoire
- Lectures d'anciens formats nécessaires
- Migrations bidirectionnelles (upgrade/downgrade)

**Versions récentes** :
- v42 : Format actuel stable
- v43 : Dernière version avec ajout géométrie PSM (openings)

**Localisation** : `packages/measurements/src/transformers/`

## Types de Mesures

### Particules

**Données** :
```typescript
interface ParticleMeasurement {
  siteEntityId: string;
  pointNumber: number;                 // Point de mesure
  timestamp: Date;
  values: {
    '0.3': number;                     // Particules ≥ 0.3µm par m³
    '0.5': number;                     // Particules ≥ 0.5µm par m³
    '1.0': number;                     // Particules ≥ 1.0µm par m³
    '5.0': number;                     // Particules ≥ 5.0µm par m³
  };
  deviceId: string;
  isConform?: boolean;                 // Calculé
}
```

**Calculs** :
- Conformité par taille de particule
- Statistiques (min, max, moyenne) par point
- Comparaison avec seuils normatifs

### Microbiologie

#### Air

```typescript
interface MicrobiologicalAirMeasurement {
  siteEntityId: string;
  pointNumber: number;
  method: 'active' | 'passive';
  timestamp: Date;
  value: number;                       // UFC/m³ (actif) ou UFC/boîte (passif)
  volumeSampled?: number;              // m³ (méthode active)
  exposureTime?: number;               // minutes (méthode passive)
  cultureMedium: string;               // Type de milieu
  incubationTemp: number;              // °C
  incubationDuration: number;          // heures
  isConform?: boolean;
}
```

#### Surfaces

```typescript
interface MicrobiologicalSurfaceMeasurement {
  siteEntityId: string;
  pointNumber: number;
  timestamp: Date;
  value: number;                       // UFC/gélose
  cultureMedium: string;
  surfaceType: string;                 // Mur, sol, plan de travail, etc.
  isConform?: boolean;
}
```

### Aéraulique

#### Vitesse d'Air

```typescript
interface AirVelocityMeasurement {
  siteEntityId: string;
  pointNumber: number;
  timestamp: Date;
  value: number;                       // m/s
  deviceId: string;
  isConform?: boolean;
}
```

**Calcul de conformité** :
```typescript
// Plage de conformité (ex: flux laminaire)
const minVelocity = 0.36;  // m/s
const maxVelocity = 0.54;  // m/s

isConform = value >= minVelocity && value <= maxVelocity;
```

#### Pression Différentielle

```typescript
interface PressureMeasurement {
  siteEntityId: string;
  referenceRoomId: string;             // Zone de référence
  timestamp: Date;
  value: number;                       // Pa (Pascals)
  isConform?: boolean;
}
```

#### Température et Humidité

```typescript
interface TemperatureHumidityMeasurement {
  siteEntityId: string;
  timestamp: Date;
  temperature: number;                 // °C
  humidity: number;                    // % HR
  deviceId: string;
}
```

### Taux de Brassage

**Formule scientifique** :
```typescript
/**
 * Calcule le taux de brassage d'une salle.
 *
 * Formule : Taux = (Débit soufflage / Volume salle) * 60
 *
 * @param flowRate - Débit de soufflage en m³/h
 * @param roomVolume - Volume de la salle en m³
 * @returns Taux de brassage en volumes/heure
 *
 * Référence : Norme XP X-15-206
 */
export function calculateAirChangeRate(
  flowRate: number,
  roomVolume: number
): number {
  if (roomVolume <= 0) {
    throw new Error('Room volume must be positive');
  }
  return (flowRate / roomVolume) * 60;
}
```

## Transformateurs de Formats

### Pattern de Migration

**Fichier** : `packages/measurements/src/transformers/format-transformers.ts`

```typescript
// Type version 42
export interface MeasurementDataV42 {
  version: 42;
  particleMeasurements: ParticleMeasurementV42[];
  microMeasurements: MicroMeasurementV42[];
  aeroMeasurements: AeroMeasurementV42[];
}

// Type version 43 (nouveau champ openings)
export interface MeasurementDataV43 {
  version: 43;
  particleMeasurements: ParticleMeasurementV43[];
  microMeasurements: MicroMeasurementV43[];
  aeroMeasurements: AeroMeasurementV43[];
  openings: OpeningGeometry[];         // NOUVEAU
}

// Migration forward v42 → v43
export function transformV42ToV43(data: MeasurementDataV42): MeasurementDataV43 {
  return {
    version: 43,
    particleMeasurements: data.particleMeasurements.map(transformParticle),
    microMeasurements: data.microMeasurements.map(transformMicro),
    aeroMeasurements: data.aeroMeasurements.map(transformAero),
    openings: [],                      // Valeur par défaut
  };
}

// Migration backward v43 → v42 (rollback possible)
export function transformV43ToV42(data: MeasurementDataV43): MeasurementDataV42 {
  return {
    version: 42,
    particleMeasurements: data.particleMeasurements.map(rollbackParticle),
    microMeasurements: data.microMeasurements.map(rollbackMicro),
    aeroMeasurements: data.aeroMeasurements.map(rollbackAero),
    // Champ openings perdu (acceptable pour rollback)
  };
}
```

### Registry de Versions

```typescript
export const FORMAT_TRANSFORMERS = {
  41: {
    upgrade: transformV41ToV42,
    downgrade: transformV42ToV41,
  },
  42: {
    upgrade: transformV42ToV43,
    downgrade: transformV43ToV42,
  },
  // ... autres versions
};

export function transformToLatest(data: MeasurementData): MeasurementDataV43 {
  let current = data;
  while (current.version < 43) {
    const transformer = FORMAT_TRANSFORMERS[current.version];
    current = transformer.upgrade(current);
  }
  return current as MeasurementDataV43;
}
```

## Calculs de Conformité

### Pattern de Calcul

```typescript
export function calculateParticleConformity(
  measurements: ParticleMeasurement[],
  threshold: number
): boolean {
  const maxValue = Math.max(...measurements.map(m => m.value));
  return maxValue <= threshold;
}

export function calculateMicrobiologicalConformity(
  measurements: MicrobiologicalMeasurement[],
  threshold: number,
  method: 'max' | 'average'
): boolean {
  const values = measurements.map(m => m.value);

  if (method === 'max') {
    return Math.max(...values) <= threshold;
  } else {
    const average = values.reduce((sum, v) => sum + v, 0) / values.length;
    return average <= threshold;
  }
}
```

## Statistiques

### Calculs Statistiques

**Fichier** : `packages/measurements/src/calculations/statistics.ts`

```typescript
export function calculateStatistics(values: number[]): {
  mean: number;
  median: number;
  stdDev: number;
  min: number;
  max: number;
} {
  const sorted = [...values].sort((a, b) => a - b);
  const mean = values.reduce((sum, v) => sum + v, 0) / values.length;
  const median = sorted[Math.floor(sorted.length / 2)];

  const variance = values.reduce(
    (sum, v) => sum + Math.pow(v - mean, 2),
    0
  ) / values.length;

  const stdDev = Math.sqrt(variance);

  return {
    mean,
    median,
    stdDev,
    min: sorted[0],
    max: sorted[sorted.length - 1],
  };
}
```

## Snapshots de Session

### Structure de Snapshot

```typescript
interface MeasurementSessionSnapshot {
  version: number;                     // Version de format (43)
  auditId: string;
  sessionId: string;
  timestamp: Date;

  // Toutes les mesures de la session
  particleMeasurements: ParticleMeasurement[];
  microAirMeasurements: MicrobiologicalAirMeasurement[];
  microSurfaceMeasurements: MicrobiologicalSurfaceMeasurement[];
  aeroMeasurements: AeroMeasurement[];

  // Métadonnées
  devices: DeviceSnapshot[];
  siteEntities: SiteEntitySnapshot[];

  // État de la session
  status: 'in-progress' | 'completed';
  completedAt?: Date;
}
```

**Problématique** : Snapshots volumineux (1-5 MB) stockés en MongoDB

**Solution recommandée** : Migration vers GCS (Google Cloud Storage)

## Génération de Conclusions

### Pattern de Conclusion

**Basé sur** : Fichier Excel `Avis_interpretation_2025-10-20.xlsx`

```typescript
export function generateMeasurementConclusion(
  testType: string,
  isConform: boolean,
  value: number,
  threshold: number,
  normReference: string
): string {
  if (testType === 'particle-counting') {
    if (isConform) {
      return `Les résultats de comptage particulaire sont conformes aux exigences de la norme ${normReference}.`;
    } else {
      return `Les résultats de comptage particulaire révèlent un dépassement des limites autorisées par la norme ${normReference} (${value} vs ${threshold}).`;
    }
  }

  if (testType === 'microbiological-air') {
    // Logique spécifique microbiologie
  }

  // ... autres types de tests
}
```

## Géométrie PSM (v43)

### Nouveau Champ Openings

**Ajouté en v43** pour gérer les ouvertures de PSM

```typescript
interface OpeningGeometry {
  siteEntityId: string;
  openings: Array<{
    position: 'front' | 'top' | 'side';
    width: number;                     // mm
    height: number;                    // mm
    isFiltered: boolean;
  }>;
}
```

**Usage** : Calculs spécifiques pour tests de confinement PSM

## Exports du Package

### Types

```typescript
export interface ParticleMeasurement { }
export interface MicrobiologicalAirMeasurement { }
export interface MicrobiologicalSurfaceMeasurement { }
export interface AirVelocityMeasurement { }
export interface PressureMeasurement { }
export interface MeasurementSessionSnapshot { }
```

### Fonctions

```typescript
// Calculs de conformité
export function calculateParticleConformity(...): boolean { }
export function calculateMicrobiologicalConformity(...): boolean { }

// Statistiques
export function calculateStatistics(...): Statistics { }

// Transformateurs
export function transformV42ToV43(...): MeasurementDataV43 { }
export function transformV43ToV42(...): MeasurementDataV42 { }
export function transformToLatest(...): MeasurementDataV43 { }

// Calculs scientifiques
export function calculateAirChangeRate(...): number { }

// Conclusions
export function generateMeasurementConclusion(...): string { }
```

## Tests Unitaires

**Couverture minimale** : 90%

**Focus sur** :
- Transformateurs de formats (v1→v43)
- Calculs scientifiques (formules précises)
- Calculs de conformité

**Exemple** :
```typescript
describe('calculateAirChangeRate', () => {
  it('should calculate correctly', () => {
    expect(calculateAirChangeRate(1000, 50)).toBe(1200);
  });

  it('should throw if volume is zero', () => {
    expect(() => calculateAirChangeRate(1000, 0)).toThrow();
  });
});
```

## Évolutions Futures

### Migration Snapshots

**Problème actuel** : Snapshots volumineux en MongoDB (1-5 MB)

**Solution recommandée** :
1. Déplacer snapshots vers GCS
2. Stocker seulement URL en MongoDB
3. Compression JSON.gz
4. URLs signées pour sécurité

**Impact** : Performance améliorée, scalabilité

### Simplification des Formats

**Objectif long terme** : Réduire de 43 à ~10 versions actives

**Méthode** : Migration de données anciennes

## Résumé

1. **43 versions** de formats (v1→v43)
2. **Transformateurs** bidirectionnels obligatoires
3. **Types de mesures** : Particules, Micro (air + surfaces), Aéraulique
4. **Calculs** : Conformité, statistiques, scientifiques
5. **Snapshots** : État complet de session (volumineux)
6. **Conclusions** : Génération automatique standardisée
7. **Principe** : Logique métier pure, testable à 90%+
8. **Évolution** : Migration snapshots vers GCS recommandée
