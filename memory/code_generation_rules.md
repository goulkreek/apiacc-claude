# Règles de Génération de Code et Templates

## Principes Généraux

### Respect des Conventions

**Toujours** suivre les conventions documentées dans :
- `conventions_typescript.md`
- `backend_patterns.md`
- `frontend_patterns.md`
- `graphql_schemas.md`
- `mongoose_models.md`

### Pas de Duplication

**Avant de créer** :
1. Rechercher si un fichier similaire existe
2. Copier le pattern existant
3. Adapter au nouveau contexte

### Patterns > Code Custom

**Privilégier** les patterns éprouvés du projet plutôt que des solutions nouvelles

## Templates Backend

### Nouveau Module NestJS

**Structure à créer** :
```
packages/backend/src/model/[entity]/
├── [entity].schema.ts
├── [entity].graphql
├── [entity].service.ts
├── [entity].resolver.ts
├── [entity].transformer.ts
├── [entity].dataloader.ts
├── [entity].module.ts
└── [entity].spec.ts
```

### Template Schema Mongoose

```typescript
import { Schema, Prop, SchemaFactory } from '@nestjs/mongoose';
import { Document, SchemaTypes, ObjectId } from 'mongoose';

@Schema({ collection: '[entities]', timestamps: true })
export class MG[Entity] {
  @Prop({ required: true })
  name: string;

  @Prop({ type: SchemaTypes.ObjectId, ref: 'MGRelated', required: true })
  relatedId: ObjectId;

  @Prop({ type: Date, default: Date.now })
  createdAt: Date;

  @Prop({ type: Date, default: Date.now })
  updatedAt: Date;
}

export type [Entity]Document = MG[Entity] & Document;
export const [Entity]Schema = SchemaFactory.createForClass(MG[Entity]);

// Indexes
[Entity]Schema.index({ relatedId: 1, createdAt: -1 });
```

### Template Service

```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { ApolloError } from 'apollo-server-express';

@Injectable()
export class [Entity]Service {
  constructor(
    @InjectModel(MG[Entity].name)
    private [entity]Model: Model<[Entity]Document>,
  ) {}

  async findById(id: string): Promise<[Entity]Document | null> {
    return this.[entity]Model.findById(id).exec();
  }

  async findAll(): Promise<[Entity]Document[]> {
    return this.[entity]Model.find().sort({ createdAt: -1 }).exec();
  }

  async create(input: Create[Entity]Input): Promise<[Entity]Document> {
    const entity = new this.[entity]Model(input);
    return entity.save();
  }

  async update(
    id: string,
    input: Update[Entity]Input
  ): Promise<[Entity]Document> {
    const entity = await this.[entity]Model.findById(id);

    if (!entity) {
      throw new ApolloError('[Entity] not found', 'NOT_FOUND');
    }

    Object.assign(entity, input);
    return entity.save();
  }

  async delete(id: string): Promise<boolean> {
    const result = await this.[entity]Model.deleteOne({ _id: id });
    return result.deletedCount > 0;
  }
}
```

### Template Resolver

```typescript
import { Resolver, Query, Mutation, Args, ResolveField, Parent } from '@nestjs/graphql';
import { UseGuards } from '@nestjs/common';

@Resolver(() => [Entity])
export class [Entity]Resolver {
  constructor(
    private [entity]Service: [Entity]Service,
    private relatedLoader: RelatedDataLoader,
  ) {}

  @Query(() => [Entity], { nullable: true })
  @UseGuards(JwtAuthGuard)
  async [entity](@Args('id') id: string): Promise<[Entity] | null> {
    const doc = await this.[entity]Service.findById(id);
    return doc ? to[Entity](doc) : null;
  }

  @Query(() => [[Entity]!]!)
  @UseGuards(JwtAuthGuard)
  async [entities](): Promise<[Entity][]> {
    const docs = await this.[entity]Service.findAll();
    return docs.map(to[Entity]);
  }

  @Mutation(() => [Entity])
  @UseGuards(JwtAuthGuard)
  async create[Entity](
    @Args('input') input: Create[Entity]Input,
    @CurrentUser() user: User,
  ): Promise<[Entity]> {
    const doc = await this.[entity]Service.create(input);
    return to[Entity](doc);
  }

  @Mutation(() => [Entity])
  @UseGuards(JwtAuthGuard)
  async update[Entity](
    @Args('id') id: string,
    @Args('input') input: Update[Entity]Input,
  ): Promise<[Entity]> {
    const doc = await this.[entity]Service.update(id, input);
    return to[Entity](doc);
  }

  @Mutation(() => Boolean)
  @UseGuards(JwtAuthGuard)
  async delete[Entity](@Args('id') id: string): Promise<boolean> {
    return this.[entity]Service.delete(id);
  }

  @ResolveField(() => Related)
  async related(@Parent() entity: [Entity]): Promise<Related> {
    return this.relatedLoader.load(entity.relatedId);
  }
}
```

### Template DataLoader

```typescript
import { Injectable, Scope } from '@nestjs/common';
import DataLoader from 'dataloader';

@Injectable({ scope: Scope.REQUEST })
export class [Entity]DataLoader {
  private readonly loader: DataLoader<string, [Entity]>;

  constructor(private [entity]Service: [Entity]Service) {
    this.loader = new DataLoader<string, [Entity]>(
      async (ids: readonly string[]) => {
        const entities = await this.[entity]Service.findByIds([...ids]);

        const entityMap = new Map(entities.map(e => [e.id, e]));
        return ids.map(id => entityMap.get(id) ?? null);
      }
    );
  }

  async load(id: string): Promise<[Entity]> {
    return this.loader.load(id);
  }

  async loadMany(ids: string[]): Promise<[Entity][]> {
    return this.loader.loadMany(ids);
  }
}
```

### Template Transformer

```typescript
export function to[Entity](doc: [Entity]Document): [Entity] {
  return {
    id: doc._id.toString(),
    name: doc.name,
    relatedId: doc.relatedId.toString(),
    createdAt: doc.createdAt.toISOString(),
    updatedAt: doc.updatedAt.toISOString(),
  };
}

export function to[Entities](docs: [Entity]Document[]): [Entity][] {
  return docs.map(to[Entity]);
}
```

### Template GraphQL Schema

```graphql
type [Entity] {
  id: ID!
  name: String!
  related: Related!
  createdAt: DateTime!
  updatedAt: DateTime!
}

input Create[Entity]Input {
  name: String!
  relatedId: ID!
}

input Update[Entity]Input {
  name: String
}

type Query {
  [entity](id: ID!): [Entity]
  [entities]: [[Entity]!]!
}

type Mutation {
  create[Entity](input: Create[Entity]Input!): [Entity]!
  update[Entity](id: ID!, input: Update[Entity]Input!): [Entity]!
  delete[Entity](id: ID!): Boolean!
}
```

### Template Module

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forFeature([
      { name: MG[Entity].name, schema: [Entity]Schema }
    ]),
  ],
  providers: [
    [Entity]Service,
    [Entity]Resolver,
    [Entity]DataLoader,
  ],
  exports: [[Entity]Service],
})
export class [Entity]Module {}
```

## Templates Frontend

### Template Composant React

```typescript
import React from 'react';
import { observer } from 'mobx-react-lite';
import { Card, Button } from 'antd';

interface [Component]Props {
  data: DataType;
  onAction: (id: string) => void;
}

export const [Component]: React.FC<[Component]Props> = observer(({
  data,
  onAction,
}) => {
  const handleClick = () => {
    onAction(data.id);
  };

  return (
    <Card title={data.name}>
      <p>{data.description}</p>
      <Button type="primary" onClick={handleClick}>
        Action
      </Button>
    </Card>
  );
});
```

### Template Formulaire

```typescript
import React from 'react';
import { useForm, Controller } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';
import { Form, Input, Button } from 'antd';

const schema = yup.object({
  name: yup.string().required('Le nom est obligatoire'),
  email: yup.string().email('Email invalide').required('L\'email est obligatoire'),
});

type FormData = yup.InferType<typeof schema>;

interface [Component]FormProps {
  onSubmit: (data: FormData) => Promise<void>;
  initialValues?: FormData;
}

export const [Component]Form: React.FC<[Component]FormProps> = ({
  onSubmit,
  initialValues,
}) => {
  const { control, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: yupResolver(schema),
    defaultValues: initialValues || { name: '', email: '' },
  });

  return (
    <Form onFinish={handleSubmit(onSubmit)} layout="vertical">
      <Form.Item
        label="Nom"
        validateStatus={errors.name ? 'error' : ''}
        help={errors.name?.message}
        required
      >
        <Controller
          name="name"
          control={control}
          render={({ field }) => <Input {...field} />}
        />
      </Form.Item>

      <Form.Item
        label="Email"
        validateStatus={errors.email ? 'error' : ''}
        help={errors.email?.message}
        required
      >
        <Controller
          name="email"
          control={control}
          render={({ field }) => <Input type="email" {...field} />}
        />
      </Form.Item>

      <Form.Item>
        <Button type="primary" htmlType="submit">
          Enregistrer
        </Button>
      </Form.Item>
    </Form>
  );
};
```

### Template Page

```typescript
import React, { useEffect } from 'react';
import { observer } from 'mobx-react-lite';
import { useStores } from '@/hooks/useStores';
import { Spin, Alert, Empty, Button } from 'antd';

export const [Page]: React.FC = observer(() => {
  const { [store]Store } = useStores();

  useEffect(() => {
    [store]Store.load();
  }, [[store]Store]);

  if ([store]Store.loading) {
    return <Spin size="large" tip="Chargement..." />;
  }

  if ([store]Store.error) {
    return (
      <Alert
        type="error"
        message="Erreur de chargement"
        description={[store]Store.error}
      />
    );
  }

  if ([store]Store.items.length === 0) {
    return (
      <Empty description="Aucune donnée">
        <Button type="primary" onClick={[store]Store.create}>
          Créer
        </Button>
      </Empty>
    );
  }

  return (
    <div>
      {/* Contenu */}
    </div>
  );
});
```

## Templates Package Métier

### Template Fonction Métier

```typescript
/**
 * Description de la fonction.
 *
 * @param param1 - Description du paramètre
 * @param param2 - Description du paramètre
 * @returns Description du retour
 *
 * @example
 * const result = myFunction(value1, value2);
 *
 * Référence : Norme XYZ
 */
export function myFunction(
  param1: ParamType,
  param2: ParamType
): ReturnType {
  // Validation
  if (!param1) {
    throw new Error('param1 is required');
  }

  // Logique métier
  const result = calculateSomething(param1, param2);

  return result;
}
```

### Template Test Unitaire

```typescript
describe('myFunction', () => {
  it('should return expected result for valid input', () => {
    const result = myFunction(validInput1, validInput2);

    expect(result).toEqual(expectedOutput);
  });

  it('should throw error for invalid input', () => {
    expect(() => myFunction(null, validInput2)).toThrow('param1 is required');
  });

  it('should handle edge cases', () => {
    const result = myFunction(edgeCaseInput1, edgeCaseInput2);

    expect(result).toBeDefined();
  });
});
```

## Règles de Substitution

### Variables à Remplacer

Dans les templates ci-dessus, remplacer :
- `[Entity]` → Nom de l'entité en PascalCase (ex: `SiteAudit`)
- `[entity]` → Nom de l'entité en camelCase (ex: `siteAudit`)
- `[entities]` → Pluriel en camelCase (ex: `siteAudits`)
- `[Component]` → Nom du composant (ex: `AuditCard`)
- `[Page]` → Nom de la page (ex: `AuditListPage`)
- `[store]` → Nom du store (ex: `siteAudit`)

### Imports à Adapter

**Toujours** ajuster les imports selon le contexte :
- Ajouter les imports nécessaires
- Supprimer les imports inutilisés
- Respecter l'ordre (React → Libs → Local)

## Checklist Génération de Code

### Avant de Générer

- [ ] Vérifier qu'un pattern similaire existe
- [ ] Lire la convention applicable
- [ ] Identifier le template à utiliser

### Pendant la Génération

- [ ] Respecter le template choisi
- [ ] Remplacer toutes les variables
- [ ] Adapter les imports
- [ ] Suivre les conventions de nommage

### Après la Génération

- [ ] Vérifier la compilation (tsc)
- [ ] Écrire les tests unitaires
- [ ] Vérifier les conventions (no cast, etc.)
- [ ] Tester manuellement si UI

## Résumé

1. ✅ **Templates** disponibles pour tous les patterns courants
2. ✅ **Conventions** strictes à respecter
3. ✅ **Recherche** de patterns existants avant création
4. ✅ **Substitution** de variables dans templates
5. ✅ **Tests unitaires** obligatoires
6. ✅ **Compilation** vérifiée
7. ✅ **Imports** adaptés au contexte
8. ✅ **Documentation** pour fonctions métier complexes
9. ✅ **Cohérence** avec l'existant prioritaire
10. ✅ **Simplicité** > complexité inutile
