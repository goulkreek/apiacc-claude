# Conventions TypeScript du Projet APIACC

## Règles Absolues

### 1. Interdiction Totale des Casts TypeScript

❌ **INTERDIT** :
```typescript
// JAMAIS utiliser 'as'
const value = something as SomeType;

// JAMAIS utiliser '<Type>'
const value = <SomeType>something;

// JAMAIS utiliser 'any'
const value: any = something;
```

✅ **CORRECT** :
```typescript
// Utiliser le type guard
function isCustomer(obj: unknown): obj is Customer {
  return typeof obj === 'object' && obj !== null && 'customerId' in obj;
}

if (isCustomer(data)) {
  // TypeScript sait que data est Customer
  console.log(data.customerId);
}

// Ou définir des types stricts dès le départ
interface ApiResponse {
  customer: Customer;
}

const response: ApiResponse = await fetchData();
```

**Raison** : Les casts masquent les erreurs de typage et créent des bugs en production.

### 2. Typage Strict Obligatoire

✅ **Toujours typer** :
- Paramètres de fonction
- Valeurs de retour
- Propriétés d'objet
- Variables (sauf si inférence évidente)

```typescript
// ✅ BON
function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// ❌ MAUVAIS (pas de type de retour)
function calculateTotal(items: Item[]) {
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

### 3. Pas d'Any, Jamais

❌ **INTERDIT** :
```typescript
const data: any = fetchData();
function process(input: any): any { }
```

✅ **ALTERNATIVES** :
```typescript
// Utiliser unknown pour type inconnu
const data: unknown = fetchData();

// Utiliser les génériques
function process<T>(input: T): T { }

// Utiliser union types
function process(input: string | number): string | number { }
```

## Nomenclature

### Interfaces et Types

**Interfaces** : PascalCase
```typescript
interface Customer {
  id: string;
  name: string;
}

interface CreateCustomerInput {
  name: string;
  email: string;
}
```

**Types** : PascalCase
```typescript
type CustomerId = string;
type SiteAuditStatus = 'setup' | 'measurementCollect' | 'approved';
```

### Enums

**Nom** : PascalCase, suffixe `Enum`
**Valeurs** : UPPER_CASE ou PascalCase selon contexte

```typescript
// Valeurs en chaînes
enum SiteAuditStatusEnum {
  SETUP = 'setup',
  MEASUREMENT_COLLECT = 'measurementCollect',
  APPROVED = 'approved',
}

// Valeurs numériques
enum RoleEnum {
  ADMIN = 1,
  AUDITOR = 2,
  CUSTOMER = 3,
}
```

### Variables et Fonctions

**Variables** : camelCase
```typescript
const siteAudit = getSiteAudit();
let measurementSession: MeasurementSession;
```

**Fonctions** : camelCase
```typescript
function createSiteAudit(input: CreateSiteAuditInput): SiteAudit { }
async function fetchCustomers(): Promise<Customer[]> { }
```

**Constantes globales** : UPPER_CASE
```typescript
const MAX_RETRY_COUNT = 3;
const DEFAULT_PAGE_SIZE = 20;
```

## Patterns Spécifiques Backend

### Schémas Mongoose

**Préfixe MG obligatoire** :
```typescript
@Schema({ collection: 'siteaudits' })
export class MGSiteAudit {
  @Prop({ required: true })
  name: string;

  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer' })
  customerId: ObjectId;
}

export type SiteAuditDocument = MGSiteAudit & Document;
```

**Règle** : Collection en lowercase, plural

### Types GraphQL vs Mongoose

**Séparation stricte** :
```typescript
// Mongoose (préfixe MG)
class MGSiteAudit { }

// GraphQL (pas de préfixe)
class SiteAudit {
  id: string;  // Converti de ObjectId
  name: string;
}
```

**Transformation** : Fonction dédiée
```typescript
function toSiteAudit(doc: SiteAuditDocument): SiteAudit {
  return {
    id: doc._id.toString(),
    name: doc.name,
    // ... autres champs
  };
}
```

## Patterns Spécifiques Frontend

### Props de Composants React

**Interface avec suffixe Props** :
```typescript
interface SiteAuditListProps {
  audits: SiteAudit[];
  onSelect: (audit: SiteAudit) => void;
  loading?: boolean;
}

export const SiteAuditList: React.FC<SiteAuditListProps> = ({
  audits,
  onSelect,
  loading = false
}) => {
  // ...
};
```

### Stores MobX State Tree

**Suffixe Store** :
```typescript
export const SiteAuditStore = types
  .model('SiteAuditStore', {
    audits: types.array(SiteAuditModel),
    selectedId: types.maybeNull(types.string),
  })
  .views((self) => ({
    get selectedAudit(): SiteAudit | null {
      return self.audits.find(a => a.id === self.selectedId) ?? null;
    },
  }))
  .actions((self) => ({
    setSelectedId(id: string): void {
      self.selectedId = id;
    },
  }));
```

## Gestion des Nullables

### Utiliser les Opérateurs TypeScript

**Optional chaining** :
```typescript
// ✅ BON
const customerName = audit?.customer?.name;

// ❌ MAUVAIS
const customerName = audit && audit.customer && audit.customer.name;
```

**Nullish coalescing** :
```typescript
// ✅ BON
const displayName = user.nickname ?? user.name ?? 'Unknown';

// ❌ MAUVAIS
const displayName = user.nickname || user.name || 'Unknown';
```

### Définir des Types Stricts

```typescript
// ✅ BON - Explicite sur ce qui peut être null
interface User {
  id: string;
  name: string;
  nickname: string | null;  // Peut être null
  profile?: UserProfile;     // Peut être undefined
}

// ❌ MAUVAIS - any cache les problèmes
interface User {
  id: string;
  name: string;
  [key: string]: any;
}
```

## Génériques

### Utilisation Correcte

```typescript
// ✅ Fonction générique
function findById<T extends { id: string }>(items: T[], id: string): T | null {
  return items.find(item => item.id === id) ?? null;
}

// ✅ Composant générique
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (option: T) => string;
}

function Select<T>({ options, value, onChange, getLabel }: SelectProps<T>) {
  // ...
}
```

## Async/Await

### Toujours Typer les Promesses

```typescript
// ✅ BON
async function fetchCustomer(id: string): Promise<Customer> {
  const response = await api.get(`/customers/${id}`);
  return response.data;
}

// ❌ MAUVAIS (pas de type de retour)
async function fetchCustomer(id: string) {
  const response = await api.get(`/customers/${id}`);
  return response.data;
}
```

### Gestion d'Erreur

```typescript
// ✅ BON - Type guard pour erreurs
function isError(error: unknown): error is Error {
  return error instanceof Error;
}

try {
  await fetchCustomer(id);
} catch (error) {
  if (isError(error)) {
    console.error(error.message);
  } else {
    console.error('Unknown error', error);
  }
}
```

## Utility Types

### Types Dérivés

```typescript
// Partial - rendre toutes les propriétés optionnelles
type UpdateCustomerInput = Partial<Customer>;

// Pick - sélectionner certaines propriétés
type CustomerSummary = Pick<Customer, 'id' | 'name'>;

// Omit - exclure certaines propriétés
type CreateCustomerInput = Omit<Customer, 'id' | 'createdAt'>;

// Record - créer un objet indexé
type CustomerMap = Record<string, Customer>;
```

## Imports et Exports

### Organisation

```typescript
// Imports groupés par origine
import React, { useState, useEffect } from 'react';
import { observer } from 'mobx-react-lite';

import { Button, Table } from 'antd';

import { useStores } from '@/hooks/useStores';
import { Customer } from '@/types';

import { formatDate } from './utils';
```

### Exports Nommés (préférés)

```typescript
// ✅ BON - Export nommé
export function calculateTotal(items: Item[]): number { }
export const MAX_ITEMS = 100;

// ❌ ÉVITER - Export default (moins traçable)
export default function calculateTotal(items: Item[]): number { }
```

## Déclarations de Modules

### Augmentation de Types

```typescript
// Pour étendre les types de bibliothèques tierces
declare module 'some-library' {
  export interface SomeType {
    newField: string;
  }
}
```

## Documentation avec JSDoc

### Quand Documenter

**Obligatoire** :
- Fonctions métier complexes
- Calculs scientifiques
- API publiques des packages

**Optionnel** :
- Fonctions simples et évidentes
- Composants React simples

### Format

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
 * @example
 * const rate = calculateAirChangeRate(1000, 50);
 * // rate = 1200 volumes/heure
 *
 * Référence : Norme XP X-15-206
 */
export function calculateAirChangeRate(
  flowRate: number,
  roomVolume: number
): number {
  return (flowRate / roomVolume) * 60;
}
```

## Strictness TypeScript Config

### tsconfig.json Recommandé

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

## Résumé des Règles Critiques

1. ❌ **Aucun cast TypeScript** (`as`, `<Type>`)
2. ❌ **Aucun `any`**
3. ✅ **Typage strict** de toutes les fonctions et variables
4. ✅ **Préfixe MG** pour schémas Mongoose
5. ✅ **Suffixe Input** pour types GraphQL d'entrée
6. ✅ **Suffixe Props** pour props React
7. ✅ **camelCase** pour variables et fonctions
8. ✅ **PascalCase** pour types, interfaces, classes
9. ✅ **Optional chaining** (`?.`) et nullish coalescing (`??`)
10. ✅ **Type guards** au lieu de casts
