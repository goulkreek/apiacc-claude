# Règles UI/UX et Accessibilité

## Principes Généraux

### Bibliothèque UI Obligatoire

**Ant Design 5.29.1** exclusivement

❌ **INTERDIT** :
- Créer des composants UI custom (boutons, inputs, etc.)
- Utiliser d'autres bibliothèques UI
- Custom CSS pour reproduire Ant Design

✅ **AUTORISÉ** :
- Ant Design components
- Tailwind CSS pour utilities (spacing, layout)
- Custom CSS pour styles vraiment spécifiques

## Composants Ant Design

### Utilisation Obligatoire

**Formulaires** :
- `<Form>` pour structures
- `<Input>`, `<Input.TextArea>` pour texte
- `<Select>` pour listes déroulantes
- `<DatePicker>` pour dates
- `<Checkbox>`, `<Radio>` pour sélections
- `<Button>` pour actions

**Affichage de Données** :
- `<Table>` pour tableaux
- `<Descriptions>` pour détails
- `<Card>` pour cartes
- `<List>` pour listes simples

**Feedback** :
- `<Spin>` pour chargement
- `<Alert>` pour alertes
- `<message>` pour notifications toast
- `<notification>` pour notifications riches
- `<Modal>` pour dialogues

**Navigation** :
- `<Menu>` pour menus
- `<Breadcrumb>` pour fil d'ariane
- `<Tabs>` pour onglets

### Pattern Form

```typescript
<Form
  layout="vertical"                  // Labels au-dessus
  onFinish={handleSubmit}
>
  <Form.Item
    label="Nom"
    validateStatus={errors.name ? 'error' : ''}
    help={errors.name?.message}
    required
  >
    <Input placeholder="Entrez le nom" />
  </Form.Item>

  <Form.Item>
    <Button type="primary" htmlType="submit">
      Enregistrer
    </Button>
    <Button style={{ marginLeft: 8 }}>
      Annuler
    </Button>
  </Form.Item>
</Form>
```

### Pattern Table

```typescript
<Table
  dataSource={items}
  columns={columns}
  rowKey="id"
  loading={loading}
  pagination={{
    pageSize: 20,
    showSizeChanger: true,
    showTotal: (total) => `Total : ${total} éléments`,
  }}
  locale={{
    emptyText: 'Aucune donnée',
  }}
/>
```

## Langue et Traduction

### Français Obligatoire

**Toutes** les interfaces doivent être en français :

✅ **BON** :
- "Enregistrer", "Annuler", "Supprimer"
- "Créer un audit", "Modifier le client"
- Messages d'erreur en français

❌ **MAUVAIS** :
- "Save", "Cancel", "Delete"
- Messages en anglais

### Labels et Placeholders

```typescript
// ✅ BON
<Input placeholder="Entrez le nom du client" />
<Button>Créer un audit</Button>
<Empty description="Aucun audit trouvé" />

// ❌ MAUVAIS
<Input placeholder="Enter customer name" />
<Button>Create audit</Button>
<Empty description="No audits found" />
```

## Gestion des États

### Loading

**Toujours afficher** un indicateur de chargement :

```typescript
if (loading) {
  return <Spin size="large" tip="Chargement..." />;
}
```

**Dans boutons** :
```typescript
<Button type="primary" loading={submitting}>
  Enregistrer
</Button>
```

### Erreurs

**Toujours gérer** les erreurs :

```typescript
if (error) {
  return (
    <Alert
      type="error"
      message="Erreur de chargement"
      description={error.message}
      showIcon
    />
  );
}
```

**Messages utilisateur** :
```typescript
// ✅ BON - Message clair et actionnable
message.error('Impossible de créer l\'audit. Vérifiez que tous les champs sont remplis.');

// ❌ MAUVAIS - Erreur technique brute
message.error(error.toString());
```

### États Vides

**Toujours prévoir** un état vide :

```typescript
if (items.length === 0) {
  return (
    <Empty
      description="Aucun audit"
      image={Empty.PRESENTED_IMAGE_SIMPLE}
    >
      <Button type="primary" onClick={handleCreate}>
        Créer un audit
      </Button>
    </Empty>
  );
}
```

## Feedback Utilisateur

### Messages de Succès

```typescript
message.success('Audit créé avec succès');
message.success('Modifications enregistrées');
```

### Messages d'Erreur

```typescript
message.error('Erreur lors de la sauvegarde');
message.error('Ce champ est obligatoire');
```

### Confirmations

**Actions destructives** :

```typescript
Modal.confirm({
  title: 'Confirmer la suppression',
  content: 'Êtes-vous sûr de vouloir supprimer cet audit ? Cette action est irréversible.',
  okText: 'Supprimer',
  okType: 'danger',
  cancelText: 'Annuler',
  onOk: async () => {
    await deleteAudit(id);
    message.success('Audit supprimé');
  },
});
```

## Accessibilité (WCAG 2.1 Niveau AA)

### Contraste

**Ratio minimum** :
- Texte normal : 4.5:1
- Texte large (≥18pt) : 3:1
- Éléments UI : 3:1

**Ant Design** respecte ces ratios par défaut

### Navigation Clavier

**Règles** :
- Tous les éléments interactifs accessibles au Tab
- Ordre de tabulation logique
- Focus visible (Ant Design le gère)

```typescript
// ✅ BON - Navigation clavier fonctionnelle
<Button onClick={handleClick}>Action</Button>

// ❌ MAUVAIS - Div non accessible au clavier
<div onClick={handleClick}>Action</div>
```

### ARIA Labels

**Boutons avec icône seule** :

```typescript
// ✅ BON
<Button
  icon={<DeleteOutlined />}
  aria-label="Supprimer l'audit"
/>

// ❌ MAUVAIS - Pas de label textuel
<Button icon={<DeleteOutlined />} />
```

**Images** :

```typescript
// ✅ BON
<img src={logo} alt="Logo APIACC" />

// ❌ MAUVAIS
<img src={logo} />
```

### Landmarks ARIA

```typescript
<header role="banner">
  <nav role="navigation">...</nav>
</header>

<main role="main">
  <section aria-label="Liste des audits">...</section>
</main>

<footer role="contentinfo">...</footer>
```

## Responsive Design

### Breakpoints

**Ant Design breakpoints** :
- xs: < 576px
- sm: ≥ 576px
- md: ≥ 768px
- lg: ≥ 992px
- xl: ≥ 1200px
- xxl: ≥ 1600px

### Grid System

```typescript
import { Row, Col } from 'antd';

<Row gutter={16}>
  <Col xs={24} sm={12} md={8}>
    Colonne 1
  </Col>
  <Col xs={24} sm={12} md={8}>
    Colonne 2
  </Col>
  <Col xs={24} sm={24} md={8}>
    Colonne 3
  </Col>
</Row>
```

### Tables Responsive

```typescript
<Table
  scroll={{ x: 800 }}  // Scroll horizontal sur mobile
  responsive
/>
```

## Cohérence Visuelle

### Espacements

**Utiliser** les spacing Ant Design :

```typescript
<div style={{ marginBottom: 16 }}>...</div>  // Standard
<div style={{ padding: 24 }}>...</div>       // Card padding
```

**Ou** Tailwind utilities :

```typescript
<div className="mb-4 p-6">...</div>
```

### Couleurs

**Utiliser** le thème Ant Design :

```typescript
// Couleurs primaires (bleu par défaut)
<Button type="primary">Action</Button>

// Couleurs sémantiques
<Alert type="success" />  // Vert
<Alert type="error" />    // Rouge
<Alert type="warning" />  // Orange
<Alert type="info" />     // Bleu

// Tags
<Tag color="blue">Statut</Tag>
<Tag color="green">Approuvé</Tag>
<Tag color="red">Rejeté</Tag>
```

### Typographie

**Ant Design Typography** :

```typescript
import { Typography } from 'antd';
const { Title, Paragraph, Text } = Typography;

<Title level={1}>Titre principal</Title>
<Title level={2}>Sous-titre</Title>
<Paragraph>Texte de paragraphe...</Paragraph>
<Text type="secondary">Texte secondaire</Text>
<Text type="danger">Texte d'erreur</Text>
```

## UX Patterns

### Navigation

**Breadcrumb** pour navigation profonde :

```typescript
<Breadcrumb>
  <Breadcrumb.Item>
    <Link to="/">Accueil</Link>
  </Breadcrumb.Item>
  <Breadcrumb.Item>
    <Link to="/audits">Audits</Link>
  </Breadcrumb.Item>
  <Breadcrumb.Item>Audit #123</Breadcrumb.Item>
</Breadcrumb>
```

**Menu** pour navigation principale :

```typescript
<Menu mode="horizontal" selectedKeys={[currentPage]}>
  <Menu.Item key="audits" icon={<FileTextOutlined />}>
    Audits
  </Menu.Item>
  <Menu.Item key="customers" icon={<UserOutlined />}>
    Clients
  </Menu.Item>
</Menu>
```

### Formulaires

**Validation inline** :

```typescript
<Form.Item
  label="Email"
  validateStatus={errors.email ? 'error' : ''}
  help={errors.email?.message}
>
  <Input type="email" />
</Form.Item>
```

**Champs obligatoires** :

```typescript
<Form.Item
  label="Nom"
  required
  rules={[{ required: true, message: 'Le nom est obligatoire' }]}
>
  <Input />
</Form.Item>
```

### Actions

**Boutons primaires** pour action principale :

```typescript
<Button type="primary">Enregistrer</Button>
<Button>Annuler</Button>
<Button danger>Supprimer</Button>
```

**Groupement** :

```typescript
<Space>
  <Button type="primary">Enregistrer</Button>
  <Button>Annuler</Button>
</Space>
```

## Performance UX

### Lazy Loading

```typescript
import { lazy, Suspense } from 'react';

const HeavyComponent = lazy(() => import('./HeavyComponent'));

<Suspense fallback={<Spin />}>
  <HeavyComponent />
</Suspense>
```

### Debounce (recherche)

```typescript
import { debounce } from 'lodash';

const handleSearch = debounce((value: string) => {
  // Recherche
}, 300);

<Input.Search onSearch={handleSearch} />
```

## Erreurs Fréquentes à Éviter

❌ **Ne PAS** :
- Créer des composants custom pour remplacer Ant Design
- Utiliser du CSS custom pour tout
- Oublier les états loading/error/empty
- Messages d'erreur en anglais
- Boutons sans aria-label (si icône seule)
- Ignorer le responsive
- Actions destructives sans confirmation

✅ **TOUJOURS** :
- Utiliser Ant Design en priorité
- Gérer loading, error, empty
- Messages en français
- Labels clairs et accessibles
- Tester au clavier
- Tester sur mobile (responsive)
- Confirmer les suppressions

## Checklist UX

### Pour Chaque Nouvelle Page

- [ ] Loading state affiché pendant chargement
- [ ] Error state avec message clair si erreur
- [ ] Empty state si aucune donnée
- [ ] Responsive (test mobile, tablette, desktop)
- [ ] Navigation clavier fonctionnelle
- [ ] ARIA labels sur éléments interactifs
- [ ] Contraste respecté (WCAG AA)
- [ ] Messages en français
- [ ] Confirmations pour actions destructives
- [ ] Feedback utilisateur (message success/error)

## Résumé

1. ✅ **Ant Design** exclusivement pour UI
2. ✅ **Français** pour tous les textes
3. ✅ **Loading/Error/Empty** systématiquement
4. ✅ **Accessibilité** WCAG 2.1 AA
5. ✅ **Responsive** mobile/tablette/desktop
6. ✅ **Feedback** utilisateur clair et actionnable
7. ✅ **Navigation** clavier fonctionnelle
8. ✅ **Confirmations** pour actions destructives
9. ✅ **ARIA labels** pour accessibilité
10. ✅ **Cohérence** visuelle (spacing, couleurs, typo)
