# Patterns Frontend (React + MobX + Ant Design)

## Architecture Frontend

### Structure du Projet Backoffice

```
packages/backoffice/src/
├── pages/                 # Pages de routage (React Router)
├── containers/            # Composants avec logique métier
├── views/                 # Composants réutilisables avec logique
├── components/            # Composants UI purs (presentational)
├── store/                 # MobX State Tree stores
├── services/              # Services (GraphQL, API)
├── hooks/                 # Custom hooks React
├── utils/                 # Fonctions utilitaires
└── types/                 # Types TypeScript partagés
```

### Hiérarchie des Composants

**Pages** → **Containers** → **Views** → **Components**

- **Pages** : Composants de routage, orchestration
- **Containers** : Logique complexe, connexion au store
- **Views** : Logique métier, réutilisables
- **Components** : UI pure, pas de logique

## Composants React

### Pattern de Composant Fonctionnel

```typescript
import React from 'react';
import { observer } from 'mobx-react-lite';

interface SiteAuditCardProps {
  audit: SiteAudit;
  onSelect: (audit: SiteAudit) => void;
  showActions?: boolean;
}

export const SiteAuditCard: React.FC<SiteAuditCardProps> = observer(({
  audit,
  onSelect,
  showActions = true,
}) => {
  const handleClick = () => {
    onSelect(audit);
  };

  return (
    <div onClick={handleClick}>
      <h3>{audit.name}</h3>
      <p>Status: {audit.status}</p>
      {showActions && <button>View</button>}
    </div>
  );
});
```

### Règles Obligatoires

1. **observer()** wrapper pour tous les composants utilisant MobX
2. **Props interface** avec suffixe `Props`
3. **React.FC<PropsType>** pour typage
4. **Props destructuring** avec valeurs par défaut
5. **Nommage** : PascalCase, export nommé

## MobX State Tree

### Pattern de Store

```typescript
import { types } from 'mobx-state-tree';

export const SiteAuditModel = types.model('SiteAudit', {
  id: types.identifier,
  name: types.string,
  status: types.enumeration(['setup', 'measurementCollect', 'approved']),
  customerId: types.string,
});

export const SiteAuditStore = types
  .model('SiteAuditStore', {
    audits: types.array(SiteAuditModel),
    selectedId: types.maybeNull(types.string),
    loading: types.optional(types.boolean, false),
    error: types.maybeNull(types.string),
  })
  .views((self) => ({
    get selectedAudit(): typeof SiteAuditModel.Type | null {
      return self.audits.find(a => a.id === self.selectedId) ?? null;
    },

    get sortedAudits(): typeof SiteAuditModel.Type[] {
      return self.audits.slice().sort((a, b) =>
        a.name.localeCompare(b.name)
      );
    },
  }))
  .actions((self) => ({
    setSelectedId(id: string | null): void {
      self.selectedId = id;
    },

    setLoading(loading: boolean): void {
      self.loading = loading;
    },

    setError(error: string | null): void {
      self.error = error;
    },

    addAudit(audit: typeof SiteAuditModel.Type): void {
      self.audits.push(audit);
    },

    async loadAudits(): Promise<void> {
      self.setLoading(true);
      self.setError(null);

      try {
        const audits = await fetchAudits(); // Service GraphQL
        self.audits.replace(audits);
      } catch (error) {
        self.setError(error.message);
      } finally {
        self.setLoading(false);
      }
    },
  }));
```

### Hook useStores

```typescript
import { useContext } from 'react';
import { StoreContext } from '@/store/StoreContext';

export function useStores() {
  const stores = useContext(StoreContext);
  if (!stores) {
    throw new Error('useStores must be used within StoreProvider');
  }
  return stores;
}

// Utilisation dans composant
const { siteAuditStore } = useStores();
```

## Apollo Client (GraphQL)

### Queries

```typescript
import { useQuery, gql } from '@apollo/client';

const GET_AUDITS = gql`
  query GetAudits {
    siteAudits {
      id
      name
      status
      customer {
        id
        name
      }
    }
  }
`;

export const AuditList: React.FC = () => {
  const { data, loading, error } = useQuery(GET_AUDITS);

  if (loading) return <Spin />;
  if (error) return <Alert type="error" message={error.message} />;

  return (
    <div>
      {data.siteAudits.map(audit => (
        <AuditCard key={audit.id} audit={audit} />
      ))}
    </div>
  );
};
```

### Mutations

```typescript
import { useMutation, gql } from '@apollo/client';

const CREATE_AUDIT = gql`
  mutation CreateAudit($input: CreateSiteAuditInput!) {
    createSiteAudit(input: $input) {
      id
      name
      status
    }
  }
`;

export const CreateAuditForm: React.FC = () => {
  const [createAudit, { loading }] = useMutation(CREATE_AUDIT, {
    refetchQueries: [{ query: GET_AUDITS }],
    onCompleted: () => {
      message.success('Audit créé avec succès');
    },
    onError: (error) => {
      message.error(`Erreur: ${error.message}`);
    },
  });

  const handleSubmit = (values: CreateAuditInput) => {
    createAudit({ variables: { input: values } });
  };

  return <Form onFinish={handleSubmit}>...</Form>;
};
```

## Formulaires (React Hook Form + Yup)

### Pattern de Formulaire

```typescript
import { useForm, Controller } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';
import { Form, Input, Button, Select } from 'antd';

const schema = yup.object({
  name: yup.string().required('Le nom est obligatoire'),
  customerId: yup.string().required('Le client est obligatoire'),
  description: yup.string().optional(),
});

type FormData = yup.InferType<typeof schema>;

export const SiteAuditForm: React.FC = () => {
  const { control, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: yupResolver(schema),
    defaultValues: {
      name: '',
      customerId: '',
      description: '',
    },
  });

  const onSubmit = async (data: FormData) => {
    try {
      await createSiteAudit(data);
      message.success('Audit créé');
    } catch (error) {
      message.error(`Erreur: ${error.message}`);
    }
  };

  return (
    <Form onFinish={handleSubmit(onSubmit)} layout="vertical">
      <Form.Item
        label="Nom"
        validateStatus={errors.name ? 'error' : ''}
        help={errors.name?.message}
      >
        <Controller
          name="name"
          control={control}
          render={({ field }) => <Input {...field} />}
        />
      </Form.Item>

      <Form.Item
        label="Client"
        validateStatus={errors.customerId ? 'error' : ''}
        help={errors.customerId?.message}
      >
        <Controller
          name="customerId"
          control={control}
          render={({ field }) => (
            <Select {...field}>
              {customers.map(c => (
                <Select.Option key={c.id} value={c.id}>
                  {c.name}
                </Select.Option>
              ))}
            </Select>
          )}
        />
      </Form.Item>

      <Button type="primary" htmlType="submit">
        Enregistrer
      </Button>
    </Form>
  );
};
```

## Ant Design

### Composants Ant Design Obligatoires

**Ne JAMAIS créer de composants custom pour** :
- Boutons → `<Button>`
- Tableaux → `<Table>`
- Formulaires → `<Form>`, `<Input>`, `<Select>`
- Modales → `<Modal>`
- Notifications → `message`, `notification`
- Loading → `<Spin>`
- Alertes → `<Alert>`

### Utilisation de Table

```typescript
import { Table } from 'antd';
import type { ColumnsType } from 'antd/es/table';

interface AuditTableProps {
  audits: SiteAudit[];
  onSelect: (audit: SiteAudit) => void;
}

export const AuditTable: React.FC<AuditTableProps> = ({ audits, onSelect }) => {
  const columns: ColumnsType<SiteAudit> = [
    {
      title: 'Nom',
      dataIndex: 'name',
      key: 'name',
      sorter: (a, b) => a.name.localeCompare(b.name),
    },
    {
      title: 'Client',
      dataIndex: ['customer', 'name'],
      key: 'customer',
    },
    {
      title: 'Statut',
      dataIndex: 'status',
      key: 'status',
      render: (status: string) => <Tag color="blue">{status}</Tag>,
    },
    {
      title: 'Actions',
      key: 'actions',
      render: (_, record) => (
        <Button type="link" onClick={() => onSelect(record)}>
          Voir
        </Button>
      ),
    },
  ];

  return (
    <Table
      dataSource={audits}
      columns={columns}
      rowKey="id"
      pagination={{ pageSize: 20 }}
    />
  );
};
```

### Utilisation de Modal

```typescript
import { Modal } from 'antd';

export const DeleteConfirmModal: React.FC = () => {
  const handleDelete = () => {
    Modal.confirm({
      title: 'Confirmer la suppression',
      content: 'Êtes-vous sûr de vouloir supprimer cet audit ?',
      okText: 'Supprimer',
      okType: 'danger',
      cancelText: 'Annuler',
      onOk: async () => {
        await deleteAudit();
        message.success('Audit supprimé');
      },
    });
  };

  return <Button danger onClick={handleDelete}>Supprimer</Button>;
};
```

### Messages et Notifications

```typescript
import { message, notification } from 'antd';

// Message simple (disparaît automatiquement)
message.success('Opération réussie');
message.error('Une erreur est survenue');
message.warning('Attention');
message.info('Information');

// Notification (plus de contenu)
notification.success({
  message: 'Audit créé',
  description: 'L\'audit a été créé avec succès.',
  duration: 3,
});

notification.error({
  message: 'Erreur de sauvegarde',
  description: error.message,
});
```

## Gestion des États

### Pattern Loading/Error/Empty

```typescript
export const AuditList: React.FC = observer(() => {
  const { siteAuditStore } = useStores();

  // ✅ Toujours gérer loading
  if (siteAuditStore.loading) {
    return <Spin size="large" />;
  }

  // ✅ Toujours gérer error
  if (siteAuditStore.error) {
    return (
      <Alert
        type="error"
        message="Erreur de chargement"
        description={siteAuditStore.error}
      />
    );
  }

  // ✅ Gérer état vide
  if (siteAuditStore.audits.length === 0) {
    return (
      <Empty
        description="Aucun audit"
        image={Empty.PRESENTED_IMAGE_SIMPLE}
      >
        <Button type="primary">Créer un audit</Button>
      </Empty>
    );
  }

  // Affichage normal
  return <Table dataSource={siteAuditStore.audits} />;
});
```

## Hooks React

### Custom Hook Pattern

```typescript
import { useState, useEffect } from 'react';

export function useAudits() {
  const [audits, setAudits] = useState<SiteAudit[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    let isMounted = true;

    const fetchData = async () => {
      try {
        setLoading(true);
        const data = await fetchAudits();
        if (isMounted) {
          setAudits(data);
        }
      } catch (err) {
        if (isMounted) {
          setError(err);
        }
      } finally {
        if (isMounted) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      isMounted = false;
    };
  }, []);

  return { audits, loading, error };
}
```

## Optimisation Performance

### useMemo

```typescript
import { useMemo } from 'react';

export const AuditList: React.FC = ({ audits }) => {
  // ✅ Mémoïser les calculs coûteux
  const sortedAudits = useMemo(() => {
    return audits.slice().sort((a, b) => a.name.localeCompare(b.name));
  }, [audits]);

  const statistics = useMemo(() => {
    return calculateStatistics(audits);
  }, [audits]);

  return <div>...</div>;
};
```

### useCallback

```typescript
import { useCallback } from 'react';

export const AuditList: React.FC = () => {
  // ✅ Mémoïser les callbacks passés aux enfants
  const handleSelect = useCallback((audit: SiteAudit) => {
    console.log('Selected', audit.id);
  }, []);

  return <AuditCard audit={audit} onSelect={handleSelect} />;
};
```

### React.lazy (Code Splitting)

```typescript
import React, { Suspense, lazy } from 'react';

// ✅ Lazy load des pages lourdes
const ReportPage = lazy(() => import('./pages/ReportPage'));

export const App: React.FC = () => {
  return (
    <Suspense fallback={<Spin size="large" />}>
      <ReportPage />
    </Suspense>
  );
};
```

## Routage (React Router)

### Pattern de Routes

```typescript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';

export const AppRoutes: React.FC = () => {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Navigate to="/audits" replace />} />
        <Route path="/login" element={<LoginPage />} />
        <Route path="/audits" element={<AuditListPage />} />
        <Route path="/audits/:id" element={<AuditDetailPage />} />
        <Route path="/audits/:id/edit" element={<AuditEditPage />} />
        <Route path="*" element={<NotFoundPage />} />
      </Routes>
    </BrowserRouter>
  );
};
```

### Navigation

```typescript
import { useNavigate, useParams } from 'react-router-dom';

export const AuditDetail: React.FC = () => {
  const navigate = useNavigate();
  const { id } = useParams<{ id: string }>();

  const handleBack = () => {
    navigate('/audits');
  };

  const handleEdit = () => {
    navigate(`/audits/${id}/edit`);
  };

  return <div>...</div>;
};
```

## Tests Unitaires

### Pattern de Test

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import '@testing-library/jest-dom';

describe('SiteAuditCard', () => {
  it('should render audit name', () => {
    const audit = { id: '1', name: 'Test Audit', status: 'setup' };

    render(<SiteAuditCard audit={audit} onSelect={jest.fn()} />);

    expect(screen.getByText('Test Audit')).toBeInTheDocument();
  });

  it('should call onSelect when clicked', () => {
    const audit = { id: '1', name: 'Test Audit', status: 'setup' };
    const onSelect = jest.fn();

    render(<SiteAuditCard audit={audit} onSelect={onSelect} />);

    fireEvent.click(screen.getByText('Test Audit'));

    expect(onSelect).toHaveBeenCalledWith(audit);
  });
});
```

## Intégration avec Packages Métier

### Import Direct

```typescript
import { calculateStatistics } from '@igienairco/apiacc-measurements';
import { checkParticleConformity } from '@igienairco/apiacc-audit';

export const MeasurementView: React.FC = ({ measurements }) => {
  const stats = calculateStatistics(measurements.map(m => m.value));

  return (
    <div>
      <p>Moyenne: {stats.mean}</p>
      <p>Max: {stats.max}</p>
    </div>
  );
};
```

## Résumé des Patterns Obligatoires

1. ✅ **observer()** pour composants MobX
2. ✅ **Ant Design** exclusivement pour UI
3. ✅ **React Hook Form + Yup** pour formulaires
4. ✅ **Apollo Client** pour GraphQL
5. ✅ **MobX State Tree** pour état global
6. ✅ **Gestion Loading/Error/Empty** systématique
7. ✅ **Props interface** avec suffixe Props
8. ✅ **useMemo/useCallback** pour optimisation
9. ✅ **React.lazy** pour code splitting
10. ✅ **Tests unitaires** pour composants critiques
