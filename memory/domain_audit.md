# Domaine Métier : Audit et Conformité

## Vue d'Ensemble

**Package** : `packages/audit/`

**Rôle** : Logique métier pure pour la conformité en salles propres

**Principe** : TypeScript pur, pas de dépendances UI ou DB

## Normes et Réglementations

### ISO 14644

**Classification de propreté particulaire**

**Classes** :
- ISO 5 : Très propre (≤ 3,520 particules ≥ 0.5µm par m³)
- ISO 7 : Propre (≤ 352,000 particules ≥ 0.5µm par m³)
- ISO 8 : Moins propre (≤ 3,520,000 particules ≥ 0.5µm par m³)

**États** :
- Au repos (at rest) : Salle vide, équipement fonctionnel
- En activité (in operation) : Personnel présent, activité normale

### BPF (Bonnes Pratiques de Fabrication)

**Grades pharmaceutiques européens**

- **Grade A** : Zone critique (remplissage aseptique)
- **Grade B** : Zone de préparation et d'habillage
- **Grade C** : Zone de production
- **Grade D** : Zone de préparation moins critique

**Correspondance ISO** :
- Grade A → ISO 5 (au repos et en activité)
- Grade B → ISO 5 (repos), ISO 7 (activité)
- Grade C → ISO 7 (repos), ISO 8 (activité)
- Grade D → ISO 8

### COFRAC (Accréditation)

**Traçabilité métrologique** :
- Dispositifs calibrés et certifiés
- Incertitudes de mesure documentées
- Chaîne de traçabilité aux étalons nationaux

**Référence** : NF EN ISO/IEC 17025

## Types d'Entités Auditées

### Areas (Zones)

**Caractéristiques** :
- Surface (m²)
- Volume (m³)
- Classification (ISO ou BPF)
- État (au repos / en activité)

**Tests applicables** :
- Comptage particulaire
- Microbiologie air et surfaces
- Aéraulique (pression, taux de brassage)

### Cabinets (Équipements)

**Types** :
- **PSM** : Poste de Sécurité Microbiologique
- **HFL** : Hotte à Flux Laminaire
- **Isolateur** : Environnement isolé
- **LAF** : Local d'Air Filtré

**Tests applicables** :
- Comptage particulaire
- Vitesse d'air (flux laminaire)
- Test de confinement
- Test d'intégrité filtre

### Grills (Grilles de Ventilation)

**Types** :
- Soufflage (air entrant)
- Extraction (air sortant)

**Tests applicables** :
- Vitesse d'air
- Débit

## Types de Tests

### Comptage Particulaire

**Principe** : Mesure concentration de particules par taille

**Classes de taille** :
- 0.3 µm, 0.5 µm, 1 µm, 5 µm

**Conformité** :
```typescript
isConform = max(measurements) ≤ threshold
```

**Localisation code** : `packages/audit/src/conformity/particle-conformity.ts`

### Microbiologie

#### Air

**Méthodes** :
- **Actif** : Impaction sur gélose, résultat en UFC/m³
- **Passif** : Exposition gélose, résultat en UFC/boîte

**Conformité** :
```typescript
// Selon protocole client
isConform = moyenne(measurements) ≤ threshold
// OU
isConform = max(measurements) ≤ threshold
```

#### Surfaces

**Méthode** : Géloses contact

**Résultat** : UFC/gélose

### Aéraulique

#### Vitesse d'Air

**Principe** : Mesure vitesse en m/s (anémomètre)

**Applications** :
- Flux laminaire (PSM, HFL)
- Grilles de soufflage

**Conformité** : Plage de vitesse (ex: 0.36 - 0.54 m/s pour flux laminaire)

#### Pression Différentielle

**Principe** : Mesure différence de pression en Pa

**Conformité** : Cascade de pression (ex: ≥ 15 Pa entre zones)

#### Taux de Brassage

**Formule** :
```typescript
tauxBrassage = (débitSoufflage / volumeSalle) * 60
// Unité : volumes/heure
```

**Conformité** : Selon norme (ex: ≥ 20 volumes/h)

### Tests Spécialisés

- Test de turbulences (visualisation fumée)
- Test de confinement (PSM, isolateurs)
- Test d'intégrité filtres (DOP, PAO)

## Protocoles de Conformité

### Structure d'un Protocole

**Fichier type** : `packages/audit/src/protocols/conformity-protocol.ts`

```typescript
interface ConformityProtocol {
  normTestId: string;              // ID du test (ex: 'particle-counting')
  siteEntityTypeId: string;        // Type d'entité (ex: 'area', 'psm')
  classification: string;          // Grade A, B, C, D ou ISO 5, 7, 8
  state: 'at-rest' | 'in-operation';
  thresholds: ConformityThreshold[];
  requiredTests: string[];
  optionalTests: string[];
  cofraccApplicable: boolean;
}

interface ConformityThreshold {
  particleSize?: string;           // '0.5µm', '5µm'
  maxValue: number;
  unit: string;
  normReference: string;           // 'ISO 14644-1', 'BPF Annexe 1'
}
```

### Exemples de Protocoles

**Area ISO 5 au repos** :
- Particules 0.5µm : ≤ 3,520 p/m³
- Particules 5µm : ≤ 29 p/m³
- Microbiologie air : ≤ 1 UFC/m³
- Microbiologie surface : ≤ 1 UFC/gélose

**PSM Grade A** :
- Particules 0.5µm : ≤ 3,520 p/m³
- Vitesse d'air : 0.36 - 0.54 m/s
- Test de confinement obligatoire

## Habilitations

### Système d'Habilitations

**Principe** : Contrôler qui peut réaliser quels tests sur quels types d'entités

**Structure** :
```typescript
interface Habilitation {
  userId: string;
  normTestId: string;              // Type de test
  siteEntityTypeId?: string;       // Type d'entité (optionnel)
  validFrom: Date;
  validUntil: Date;
  certificationNumber?: string;
  cofraccAccredited: boolean;
}
```

**Validation** :
```typescript
function isUserHabilitated(
  userId: string,
  normTestId: string,
  siteEntityTypeId: string,
  testDate: Date
): boolean {
  // Vérifier habilitation valide à la date du test
  // Vérifier type de test et type d'entité
}
```

**Localisation** : `packages/audit/src/habilitations/`

## Calculs de Conformité

### Pattern de Fonction

```typescript
export interface ConformityResult {
  isConform: boolean;
  value: number;                   // Valeur mesurée (max, moyenne, etc.)
  threshold: number;
  deviation: number;               // Écart si non conforme
  conclusion: string;
}

export function checkParticleConformity(
  measurements: number[],
  threshold: ConformityThreshold
): ConformityResult {
  const maxValue = Math.max(...measurements);
  const isConform = maxValue <= threshold.maxValue;

  return {
    isConform,
    value: maxValue,
    threshold: threshold.maxValue,
    deviation: isConform ? 0 : maxValue - threshold.maxValue,
    conclusion: isConform
      ? 'Conforme'
      : `Non conforme (dépassement de ${maxValue - threshold.maxValue} ${threshold.unit})`,
  };
}
```

### Statistiques

```typescript
export function calculateStatistics(values: number[]): {
  mean: number;
  median: number;
  stdDev: number;
  min: number;
  max: number;
} {
  // Implémentation dans packages/audit/src/statistics/
}
```

## Génération de Conclusions

### Basé sur Fichier Excel

**Source** : `Avis_interpretation_2025-10-20.xlsx`

**Principe** : Conclusions automatiques selon :
- Type de test
- Résultat de conformité
- Contexte (COFRAC, type d'entité)

**Fichier** : `packages/audit/src/conclusions/auto-conclusion.ts`

```typescript
export function generateConclusion(
  testType: string,
  result: ConformityResult,
  context: TestContext
): string {
  // Logique basée sur le fichier Excel
  // Conclusions standardisées par type de test
}
```

## Règles Métier Spécifiques

### Validation Croisée

**Exemple** : PSM nécessite à la fois comptage particulaire ET test de confinement

**Exemple** : Si microbiologie > seuil → Analyse obligatoire

### Gestion des Anomalies

**Principe** : Si test non conforme, déclencher actions

**Actions possibles** :
- Commentaire obligatoire
- Re-test requis
- Blocage approbation

## Exports du Package

### Types

```typescript
export interface ConformityProtocol { }
export interface ConformityThreshold { }
export interface ConformityResult { }
export interface Habilitation { }
```

### Fonctions

```typescript
export function checkParticleConformity(...): ConformityResult { }
export function checkMicrobiologicalConformity(...): ConformityResult { }
export function getProtocolForEntity(...): ConformityProtocol | null { }
export function isUserHabilitated(...): boolean { }
export function generateConclusion(...): string { }
```

## Tests Unitaires

**Couverture minimale** : 90%

**Fichiers** : `*.spec.ts` à côté de chaque fichier source

**Exemple** :
```typescript
describe('checkParticleConformity', () => {
  it('should return conform when below threshold', () => { });
  it('should return non-conform when above threshold', () => { });
  it('should calculate deviation correctly', () => { });
});
```

## Références Normatives

- ISO 14644-1 : Classification de l'air
- ISO 14644-2 : Surveillance
- NF EN ISO 14644-3 : Méthodes de mesure
- BPF Annexe 1 : Fabrication de médicaments stériles
- NF EN ISO/IEC 17025 : Compétence des laboratoires

## Résumé

1. **Normes** : ISO 14644, BPF, COFRAC
2. **Entités** : Areas, Cabinets (PSM, HFL, Isolateur), Grills
3. **Tests** : Particules, Microbiologie, Aéraulique, Spécialisés
4. **Protocoles** : Définis par type d'entité + classification
5. **Habilitations** : Validation temporelle et par type de test
6. **Conformité** : Calculs basés sur seuils normatifs
7. **Conclusions** : Génération automatique standardisée
8. **Principe** : Logique métier pure, testable à 90%+
