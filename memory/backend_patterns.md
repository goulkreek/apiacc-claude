# Patterns Backend (NestJS + GraphQL + Mongoose)

## Architecture Modulaire NestJS

### Structure d'un Module Complet

**Localisation** : `packages/backend/src/model/[domain]/`

```
site-audit/
├── site-audit.schema.ts         # Mongoose schema (MG prefix)
├── site-audit.graphql           # GraphQL SDL
├── site-audit.service.ts        # Business logic
├── site-audit.resolver.ts       # GraphQL resolver
├── site-audit.transformer.ts    # MG → GraphQL transformations
├── site-audit.module.ts         # NestJS module definition
└── site-audit.spec.ts           # Tests unitaires
```

### Module NestJS Pattern

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: MGSiteAudit.name, schema: SiteAuditSchema }
    ]),
  ],
  providers: [
    SiteAuditService,
    SiteAuditResolver,
    SiteAuditDataLoader,
  ],
  exports: [SiteAuditService],
})
export class SiteAuditModule {}
```

## Schémas Mongoose

### Pattern de Base

**Fichier** : `[entity].schema.ts`

```typescript
import { Schema, Prop, SchemaFactory } from '@nestjs/mongoose';
import { Document, SchemaTypes, ObjectId } from 'mongoose';

@Schema({ collection: 'siteaudits', timestamps: true })
export class MGSiteAudit {
  @Prop({ required: true })
  name: string;

  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer', required: true })
  customerId: ObjectId;

  @Prop({ type: Date, default: Date.now })
  createdAt: Date;
}

export type SiteAuditDocument = MGSiteAudit & Document;
export const SiteAuditSchema = SchemaFactory.createForClass(MGSiteAudit);

// Indexes pour performance
SiteAuditSchema.index({ customerId: 1, createdAt: -1 });
SiteAuditSchema.index({ status: 1 });
```

### Règles Obligatoires

1. **Préfixe MG** pour classe Mongoose
2. **Collection** : lowercase, plural
3. **timestamps: true** pour createdAt/updatedAt automatiques
4. **Indexes** : Toujours définir pour champs de recherche/tri
5. **Type de document exporté** : `[Entity]Document`

### Relations

**ObjectId avec ref** :
```typescript
@Prop({ type: SchemaTypes.ObjectId, ref: 'MGCustomer' })
customerId: ObjectId;

@Prop({ type: [SchemaTypes.ObjectId], ref: 'MGDevice' })
deviceIds: ObjectId[];
```

**Embedded documents** :
```typescript
@Prop({ type: Object })
metadata: {
  key: string;
  value: string;
};
```

## Services NestJS

### Pattern de Service

**Fichier** : `[entity].service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { ApolloError } from 'apollo-server-express';

@Injectable()
export class SiteAuditService {
  constructor(
    @InjectModel(MGSiteAudit.name)
    private siteAuditModel: Model<SiteAuditDocument>,
  ) {}

  async findById(id: string): Promise<SiteAuditDocument | null> {
    return this.siteAuditModel.findById(id).exec();
  }

  async create(input: CreateSiteAuditInput): Promise<SiteAuditDocument> {
    // Validation métier
    if (!input.name) {
      throw new ApolloError('Name is required', 'INVALID_INPUT');
    }

    const audit = new this.siteAuditModel(input);
    return audit.save();
  }

  async update(
    id: string,
    input: UpdateSiteAuditInput
  ): Promise<SiteAuditDocument> {
    const audit = await this.siteAuditModel.findById(id);

    if (!audit) {
      throw new ApolloError('Audit not found', 'NOT_FOUND');
    }

    Object.assign(audit, input);
    return audit.save();
  }
}
```

### Gestion d'Erreur

**ApolloError pour erreurs métier** :
```typescript
throw new ApolloError('Customer not found', 'CUSTOMER_NOT_FOUND');
throw new ApolloError('Unauthorized', 'UNAUTHORIZED');
throw new ApolloError('Invalid input', 'INVALID_INPUT');
```

**Logging pour erreurs système** :
```typescript
import { Logger } from '@nestjs/common';

private readonly logger = new Logger(SiteAuditService.name);

try {
  await this.operation();
} catch (error) {
  this.logger.error('Operation failed', error);
  throw new ApolloError('Internal server error', 'INTERNAL_ERROR');
}
```

## Resolvers GraphQL

### Pattern de Resolver

**Fichier** : `[entity].resolver.ts`

```typescript
import { Resolver, Query, Mutation, Args, ResolveField, Parent } from '@nestjs/graphql';
import { UseGuards } from '@nestjs/common';

@Resolver(() => SiteAudit)
export class SiteAuditResolver {
  constructor(
    private siteAuditService: SiteAuditService,
    private customerLoader: CustomerDataLoader,
  ) {}

  @Query(() => SiteAudit, { nullable: true })
  @UseGuards(JwtAuthGuard)
  async siteAudit(@Args('id') id: string): Promise<SiteAudit | null> {
    const doc = await this.siteAuditService.findById(id);
    return doc ? toSiteAudit(doc) : null;
  }

  @Query(() => [SiteAudit])
  @UseGuards(JwtAuthGuard)
  async siteAudits(): Promise<SiteAudit[]> {
    const docs = await this.siteAuditService.findAll();
    return docs.map(toSiteAudit);
  }

  @Mutation(() => SiteAudit)
  @UseGuards(JwtAuthGuard)
  async createSiteAudit(
    @Args('input') input: CreateSiteAuditInput,
    @CurrentUser() user: User,
  ): Promise<SiteAudit> {
    const doc = await this.siteAuditService.create(input);
    return toSiteAudit(doc);
  }

  // Field Resolver avec DataLoader (OBLIGATOIRE)
  @ResolveField(() => Customer)
  async customer(@Parent() audit: SiteAudit): Promise<Customer> {
    return this.customerLoader.load(audit.customerId);
  }
}
```

### Authentification

**Guards** :
```typescript
@UseGuards(JwtAuthGuard)  // Toutes les routes sauf login
@UseGuards(RolesGuard)    // Vérification des rôles
```

**Current User** :
```typescript
import { CurrentUser } from '@/common/decorators/current-user.decorator';

async createSiteAudit(
  @Args('input') input: CreateSiteAuditInput,
  @CurrentUser() user: User,
): Promise<SiteAudit> {
  // user est l'utilisateur authentifié
}
```

## DataLoaders (OBLIGATOIRE)

### Pattern DataLoader

**Fichier** : `[entity].dataloader.ts`

```typescript
import { Injectable, Scope } from '@nestjs/common';
import DataLoader from 'dataloader';

@Injectable({ scope: Scope.REQUEST })
export class CustomerDataLoader {
  private readonly loader: DataLoader<string, Customer>;

  constructor(private customerService: CustomerService) {
    this.loader = new DataLoader<string, Customer>(
      async (ids: readonly string[]) => {
        const customers = await this.customerService.findByIds([...ids]);

        // Mapper dans le même ordre que les IDs
        const customerMap = new Map(customers.map(c => [c.id, c]));
        return ids.map(id => customerMap.get(id) ?? null);
      }
    );
  }

  async load(id: string): Promise<Customer> {
    return this.loader.load(id);
  }

  async loadMany(ids: string[]): Promise<Customer[]> {
    return this.loader.loadMany(ids);
  }
}
```

### Utilisation dans Resolver

```typescript
@ResolveField(() => Customer)
async customer(@Parent() audit: SiteAudit): Promise<Customer> {
  // ✅ BON - Utilise DataLoader (batching automatique)
  return this.customerLoader.load(audit.customerId);
}

// ❌ MAUVAIS - N+1 problem
@ResolveField(() => Customer)
async customer(@Parent() audit: SiteAudit): Promise<Customer> {
  return this.customerService.findById(audit.customerId);
}
```

**Pourquoi obligatoire ?** :
- Évite le problème N+1 (requête par élément)
- Batching automatique des requêtes
- Cache pendant la durée de la requête GraphQL

## Transformers

### Pattern de Transformation

**Fichier** : `[entity].transformer.ts`

```typescript
export function toSiteAudit(doc: SiteAuditDocument): SiteAudit {
  return {
    id: doc._id.toString(),
    name: doc.name,
    customerId: doc.customerId.toString(),
    createdAt: doc.createdAt.toISOString(),
    updatedAt: doc.updatedAt.toISOString(),
  };
}

export function toSiteAudits(docs: SiteAuditDocument[]): SiteAudit[] {
  return docs.map(toSiteAudit);
}
```

**Règle** :
- Conversion `ObjectId` → `string`
- Conversion `Date` → `ISO string`
- Séparation claire Mongoose ↔ GraphQL

## Schémas GraphQL (SDL)

### Pattern GraphQL Schema

**Fichier** : `[entity].graphql`

```graphql
type SiteAudit {
  id: ID!
  name: String!
  customer: Customer!
  site: Site!
  status: SiteAuditStatus!
  createdAt: DateTime!
  updatedAt: DateTime!
}

enum SiteAuditStatus {
  SETUP
  MEASUREMENT_COLLECT
  APPROVED
  REJECTED
}

input CreateSiteAuditInput {
  name: String!
  customerId: ID!
  siteId: ID!
}

input UpdateSiteAuditInput {
  name: String
  status: SiteAuditStatus
}

type Query {
  siteAudit(id: ID!): SiteAudit
  siteAudits(filter: SiteAuditFilter): [SiteAudit!]!
}

type Mutation {
  createSiteAudit(input: CreateSiteAuditInput!): SiteAudit!
  updateSiteAudit(id: ID!, input: UpdateSiteAuditInput!): SiteAudit!
}
```

### Conventions

- **Types** : PascalCase
- **Champs** : camelCase
- **Inputs** : Suffixe `Input`
- **Relations** : Type objet (ex: `customer: Customer!`)
- **Dates** : Scalar `DateTime`

## Validation

### Class Validator

```typescript
import { IsString, IsNotEmpty, IsOptional, IsEnum } from 'class-validator';

export class CreateSiteAuditInput {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsString()
  @IsNotEmpty()
  customerId: string;

  @IsEnum(SiteAuditStatusEnum)
  @IsOptional()
  status?: SiteAuditStatusEnum;
}
```

### Validation dans Service

```typescript
async create(input: CreateSiteAuditInput): Promise<SiteAuditDocument> {
  // Validation métier (au-delà de class-validator)
  const customer = await this.customerService.findById(input.customerId);
  if (!customer) {
    throw new ApolloError('Customer not found', 'CUSTOMER_NOT_FOUND');
  }

  const audit = new this.siteAuditModel(input);
  return audit.save();
}
```

## Tests Unitaires

### Pattern de Test

**Fichier** : `[entity].spec.ts`

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { getModelToken } from '@nestjs/mongoose';

describe('SiteAuditService', () => {
  let service: SiteAuditService;
  let model: Model<SiteAuditDocument>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        SiteAuditService,
        {
          provide: getModelToken(MGSiteAudit.name),
          useValue: {
            find: jest.fn(),
            findById: jest.fn(),
            create: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<SiteAuditService>(SiteAuditService);
    model = module.get<Model<SiteAuditDocument>>(getModelToken(MGSiteAudit.name));
  });

  describe('findById', () => {
    it('should return audit when found', async () => {
      const mockAudit = { _id: '123', name: 'Test' };
      jest.spyOn(model, 'findById').mockResolvedValue(mockAudit);

      const result = await service.findById('123');

      expect(result).toEqual(mockAudit);
      expect(model.findById).toHaveBeenCalledWith('123');
    });
  });
});
```

## Intégration avec Packages Métier

### Import Direct

```typescript
import { checkParticleConformity } from '@igienairco/apiacc-audit';
import { transformV42ToV43 } from '@igienairco/apiacc-measurements';

async function processSnapshot(snapshot: SnapshotV42): Promise<SnapshotV43> {
  return transformV42ToV43(snapshot);
}
```

**Principe** : Backend consomme les packages métier comme bibliothèques

## Optimisation des Requêtes

### Indexes MongoDB

```typescript
// Dans schema
SiteAuditSchema.index({ customerId: 1, createdAt: -1 });  // Composite
SiteAuditSchema.index({ status: 1 });                     // Simple
SiteAuditSchema.index({ 'auditor.userId': 1 });          // Nested
```

### Projection (select)

```typescript
// ✅ BON - Sélectionner uniquement les champs nécessaires
await this.siteAuditModel.find().select('name status').exec();

// ❌ ÉVITER - Charger tous les champs
await this.siteAuditModel.find().exec();
```

### Pagination

```typescript
async findPaginated(page: number, limit: number): Promise<SiteAudit[]> {
  const skip = (page - 1) * limit;
  return this.siteAuditModel.find().skip(skip).limit(limit).exec();
}
```

## Résumé des Patterns Obligatoires

1. ✅ **Structure modulaire** : Schema, Service, Resolver, Module
2. ✅ **Préfixe MG** pour Mongoose schemas
3. ✅ **DataLoaders** pour toutes les relations GraphQL
4. ✅ **Transformers** pour conversion MG → GraphQL
5. ✅ **ApolloError** pour erreurs métier
6. ✅ **Guards** pour authentification/autorisation
7. ✅ **Validation** via class-validator et logique métier
8. ✅ **Indexes** MongoDB pour performance
9. ✅ **Tests unitaires** pour tous les services
10. ✅ **Séparation stricte** : Service (logique), Resolver (exposition)
