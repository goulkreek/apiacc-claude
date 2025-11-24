# Conventions Mongoose

## Principes Généraux

### Préfixe MG Obligatoire

**Toutes** les classes Mongoose doivent avoir le préfixe `MG` :
```typescript
export class MGSiteAudit { }
export class MGCustomer { }
export class MGDevice { }
```

**Raison** : Distinction claire avec types GraphQL

### Collection Naming

**lowercase, plural** :
```typescript
@Schema({ collection: 'siteaudits' })  // ✅
@Schema({ collection: 'customers' })   // ✅

@Schema({ collection: 'SiteAudits' })  // ❌ MAUVAIS
@Schema({ collection: 'siteaudit' })   // ❌ MAUVAIS (singular)
```

## Pattern de Schéma

### Structure Standard

**Fichier** : `packages/backend/src/model/[domain]/[domain].schema.ts`

```typescript
import { Schema, Prop, SchemaFactory } from '@nestjs/mongoose';
import { Document, SchemaTypes, ObjectId } from 'mongoose';

@Schema({
  collection: 'siteaudits',
  timestamps: true  // Ajoute createdAt, updatedAt automatiquement
})
export class MGSiteAudit {
  // Champs scalaires
  @Prop({ required: true })
  name: string;

  @Prop({ required: false, default: '' })
  description: string;

  @Prop({
    type: String,
    enum: ['setup', 'measurementCollect', 'approved'],
    default: 'setup'
  })
  status: string;

  // Relations (ObjectId)
  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer', required: true })
  customerId: ObjectId;

  @Prop({ type: [SchemaTypes.ObjectId], ref: 'MGDevice', default: [] })
  deviceIds: ObjectId[];

  // Dates
  @Prop({ type: Date, default: Date.now })
  createdAt: Date;

  @Prop({ type: Date, default: Date.now })
  updatedAt: Date;
}

// Type de document (utilisé dans services)
export type SiteAuditDocument = MGSiteAudit & Document;

// Schéma Mongoose
export const SiteAuditSchema = SchemaFactory.createForClass(MGSiteAudit);

// Indexes (OBLIGATOIRE pour performance)
SiteAuditSchema.index({ customerId: 1, createdAt: -1 });
SiteAuditSchema.index({ status: 1 });
```

## Types de Propriétés

### Scalaires Simples

```typescript
// String
@Prop({ required: true })
name: string;

@Prop({ required: false, trim: true, lowercase: true })
email: string;

// Number
@Prop({ required: true, min: 0, max: 100 })
score: number;

// Boolean
@Prop({ default: false })
isActive: boolean;

// Date
@Prop({ type: Date, default: Date.now })
createdAt: Date;
```

### Enum

```typescript
@Prop({
  type: String,
  enum: ['setup', 'measurementCollect', 'approved'],
  required: true,
  default: 'setup'
})
status: string;

// Ou utiliser enum TypeScript
enum SiteAuditStatusEnum {
  SETUP = 'setup',
  MEASUREMENT_COLLECT = 'measurementCollect',
  APPROVED = 'approved',
}

@Prop({
  type: String,
  enum: Object.values(SiteAuditStatusEnum),
  default: SiteAuditStatusEnum.SETUP
})
status: string;
```

### Relations (ObjectId)

**One-to-One / Many-to-One** :
```typescript
@Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer', required: true })
customerId: ObjectId;

@Prop({ type: SchemaTypes.ObjectId, ref: 'MGUser' })
auditorId?: ObjectId;
```

**One-to-Many / Many-to-Many** :
```typescript
@Prop({ type: [SchemaTypes.ObjectId], ref: 'MGDevice', default: [] })
deviceIds: ObjectId[];
```

### Embedded Documents

**Simple Object** :
```typescript
@Prop({ type: Object })
metadata: {
  key: string;
  value: string;
};
```

**Nested Schema** :
```typescript
class Address {
  @Prop()
  street: string;

  @Prop()
  city: string;

  @Prop()
  zipCode: string;
}

@Schema()
export class MGCustomer {
  @Prop({ type: Address })
  address: Address;
}
```

### Arrays

**Array de scalaires** :
```typescript
@Prop({ type: [String], default: [] })
tags: string[];

@Prop({ type: [Number], default: [] })
measurements: number[];
```

**Array d'objets** :
```typescript
@Prop({
  type: [{
    key: String,
    value: String,
  }],
  default: []
})
properties: Array<{ key: string; value: string }>;
```

## Validations

### Built-in Validators

```typescript
// Required
@Prop({ required: true })
name: string;

// Min/Max (nombres)
@Prop({ min: 0, max: 100 })
score: number;

// MinLength/MaxLength (chaînes)
@Prop({ minlength: 3, maxlength: 100 })
username: string;

// Match (regex)
@Prop({ match: /^[a-zA-Z0-9]+$/ })
code: string;

// Enum
@Prop({ enum: ['active', 'inactive'] })
status: string;

// Trim, Lowercase, Uppercase
@Prop({ trim: true, lowercase: true })
email: string;
```

### Custom Validators

```typescript
@Prop({
  required: true,
  validate: {
    validator: function(v: string) {
      return /^[A-Z]{2}-\d{4}$/.test(v);
    },
    message: (props: any) => `${props.value} is not a valid reference!`
  }
})
reference: string;
```

## Indexes

### Simple Index

```typescript
// Index sur un seul champ
SiteAuditSchema.index({ customerId: 1 });  // Ascending
SiteAuditSchema.index({ createdAt: -1 }); // Descending
```

### Composite Index

```typescript
// Index sur plusieurs champs (ordre important)
SiteAuditSchema.index({ customerId: 1, createdAt: -1 });
SiteAuditSchema.index({ status: 1, customerId: 1 });
```

### Unique Index

```typescript
// Unicité sur un champ
SiteAuditSchema.index({ email: 1 }, { unique: true });

// Unicité composite
SiteAuditSchema.index(
  { customerId: 1, year: 1, name: 1 },
  { unique: true }
);
```

### Text Index (recherche plein texte)

```typescript
SiteAuditSchema.index({ name: 'text', description: 'text' });
```

### Sparse Index

```typescript
// Index seulement si le champ existe
SiteAuditSchema.index(
  { optionalField: 1 },
  { sparse: true }
);
```

## Timestamps

### Automatique

```typescript
@Schema({ timestamps: true })
export class MGSiteAudit {
  // createdAt et updatedAt sont ajoutés automatiquement
}
```

### Manuel

```typescript
@Schema()
export class MGSiteAudit {
  @Prop({ type: Date, default: Date.now })
  createdAt: Date;

  @Prop({ type: Date, default: Date.now })
  updatedAt: Date;
}
```

## Virtuals

### Définition

```typescript
SiteAuditSchema.virtual('fullName').get(function(this: SiteAuditDocument) {
  return `${this.firstName} ${this.lastName}`;
});

// Activer les virtuals dans toJSON
SiteAuditSchema.set('toJSON', {
  virtuals: true,
});
```

## Middleware (Hooks)

### Pre Save

```typescript
SiteAuditSchema.pre('save', function(next) {
  // Modification avant sauvegarde
  this.updatedAt = new Date();
  next();
});
```

### Pre Remove

```typescript
SiteAuditSchema.pre('remove', async function(next) {
  // Nettoyage avant suppression
  await this.model('MGRelatedEntity').deleteMany({ auditId: this._id });
  next();
});
```

## Methods et Statics

### Instance Methods

```typescript
SiteAuditSchema.methods.isApproved = function(this: SiteAuditDocument): boolean {
  return this.status === 'approved';
};

// Utilisation
const audit = await this.siteAuditModel.findById(id);
if (audit.isApproved()) { }
```

### Static Methods

```typescript
SiteAuditSchema.statics.findByCustomer = function(
  customerId: string
): Promise<SiteAuditDocument[]> {
  return this.find({ customerId }).exec();
};

// Utilisation
const audits = await this.siteAuditModel.findByCustomer(customerId);
```

## Bonnes Pratiques

### 1. Toujours Définir des Indexes

```typescript
// ✅ BON - Indexes pour recherches fréquentes
SiteAuditSchema.index({ customerId: 1, createdAt: -1 });
SiteAuditSchema.index({ status: 1 });

// ❌ MAUVAIS - Pas d'index
export const SiteAuditSchema = SchemaFactory.createForClass(MGSiteAudit);
```

### 2. Utiliser timestamps: true

```typescript
// ✅ BON
@Schema({ timestamps: true })

// ❌ ÉVITER - Gestion manuelle
@Prop({ type: Date, default: Date.now })
createdAt: Date;
```

### 3. Valeurs par Défaut

```typescript
// ✅ BON
@Prop({ type: [String], default: [] })
tags: string[];

@Prop({ type: String, default: 'setup' })
status: string;

// ❌ MAUVAIS - Pas de default (risque undefined)
@Prop({ type: [String] })
tags: string[];
```

### 4. Relations via ObjectId, pas Populate par Défaut

```typescript
// ✅ BON - ObjectId, populate via DataLoader GraphQL
@Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer' })
customerId: ObjectId;

// ❌ ÉVITER - Populate automatique (N+1 problem)
@Prop({ type: Customer, ref: 'MGCustomer', autopopulate: true })
customer: Customer;
```

## Exemples Complets

### Schéma Complet Complexe

```typescript
import { Schema, Prop, SchemaFactory } from '@nestjs/mongoose';
import { Document, SchemaTypes, ObjectId } from 'mongoose';

// Embedded schema
class Auditor {
  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGUser', required: true })
  userId: ObjectId;

  @Prop({ required: true })
  name: string;

  @Prop({ type: Date, required: true })
  assignedAt: Date;
}

// Main schema
@Schema({
  collection: 'siteaudits',
  timestamps: true,
})
export class MGSiteAudit {
  // Scalaires
  @Prop({ required: true, trim: true })
  name: string;

  @Prop({ default: '' })
  description: string;

  // Enum
  @Prop({
    type: String,
    enum: ['setup', 'measurementCollect', 'approved', 'rejected'],
    default: 'setup',
    index: true
  })
  status: string;

  // Relations
  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer', required: true })
  customerId: ObjectId;

  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGSite', required: true })
  siteId: ObjectId;

  @Prop({ type: [SchemaTypes.ObjectId], ref: 'MGDevice', default: [] })
  deviceIds: ObjectId[];

  // Embedded
  @Prop({ type: Auditor })
  auditor?: Auditor;

  // Arrays
  @Prop({ type: [String], default: [] })
  tags: string[];

  // Complex object
  @Prop({ type: Object })
  metadata?: Record<string, any>;

  // Dates (timestamps: true gère createdAt/updatedAt)
}

export type SiteAuditDocument = MGSiteAudit & Document;
export const SiteAuditSchema = SchemaFactory.createForClass(MGSiteAudit);

// Indexes
SiteAuditSchema.index({ customerId: 1, createdAt: -1 });
SiteAuditSchema.index({ siteId: 1 });
SiteAuditSchema.index({ 'auditor.userId': 1 });

// Methods
SiteAuditSchema.methods.isApproved = function(): boolean {
  return this.status === 'approved';
};

// Statics
SiteAuditSchema.statics.findByCustomer = function(customerId: string) {
  return this.find({ customerId }).sort({ createdAt: -1 }).exec();
};
```

## Résumé des Conventions

1. ✅ **Préfixe MG** obligatoire pour classes
2. ✅ **Collection** : lowercase, plural
3. ✅ **timestamps: true** pour createdAt/updatedAt auto
4. ✅ **Indexes** sur champs de recherche/tri
5. ✅ **ObjectId** pour relations (pas populate auto)
6. ✅ **Valeurs par défaut** pour arrays et optionnels
7. ✅ **Enum** pour valeurs contraintes
8. ✅ **Validation** via decorators Mongoose
9. ✅ **Type Document exporté** : `[Entity]Document`
10. ✅ **Relations résolues côté GraphQL** via DataLoaders
