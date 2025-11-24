# Guide d'Utilisation de Playwright MCP

## Configuration

Le serveur Playwright MCP est configuré dans `.mcp.json` à la racine du projet.

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    }
  }
}
```

## Activation

**IMPORTANT** : Après avoir ajouté la configuration, vous devez :
1. Redémarrer Claude Code complètement
2. Les outils MCP Playwright apparaîtront avec le préfixe `mcp__playwright__*`

## Outils Disponibles

Une fois activé, vous aurez accès aux outils suivants via les fonctions MCP :

### Navigation
- `mcp__playwright__navigate` : Aller à une URL
- `mcp__playwright__goBack` : Retour arrière
- `mcp__playwright__goForward` : Avancer
- `mcp__playwright__reload` : Recharger la page

### Interaction
- `mcp__playwright__click` : Cliquer sur un élément
- `mcp__playwright__fill` : Remplir un champ de texte
- `mcp__playwright__select` : Sélectionner une option
- `mcp__playwright__hover` : Survoler un élément
- `mcp__playwright__type` : Taper du texte

### Captures d'Écran
- `mcp__playwright__screenshot` : Capturer une screenshot
- `mcp__playwright__screenshotFullPage` : Capturer la page entière

### Network & Performance
- `mcp__playwright__emulateNetworkConditions` : Simuler réseau lent
- `mcp__playwright__setViewport` : Changer la taille de viewport
- `mcp__playwright__emulateDevice` : Émuler un appareil mobile

### Tests & Vérifications
- `mcp__playwright__waitForSelector` : Attendre qu'un élément apparaisse
- `mcp__playwright__waitForUrl` : Attendre une navigation
- `mcp__playwright__evaluate` : Exécuter du JavaScript
- `mcp__playwright__content` : Obtenir le contenu HTML

## Exemples d'Utilisation

### Test E2E Complet : Connexion Auditor

```typescript
// 1. Lancer l'application
// yarn dev (en arrière-plan)

// 2. Navigation vers login
await mcp__playwright__navigate({ url: 'http://localhost:3001/login' });

// 3. Remplir le formulaire
await mcp__playwright__fill({ selector: '[name="email"]', text: 'auditor@test.com' });
await mcp__playwright__fill({ selector: '[name="password"]', text: 'password123' });

// 4. Soumettre le formulaire
await mcp__playwright__click({ selector: 'button[type="submit"]' });

// 5. Attendre la redirection
await mcp__playwright__waitForUrl({ url: '**/audits' });

// 6. Capturer une screenshot de succès
const screenshot = await mcp__playwright__screenshot({ fullPage: false });

// 7. Vérifier qu'on est bien connecté
const content = await mcp__playwright__content();
// Vérifier que "Audits" apparaît dans le contenu
```

### Test avec Network Throttling

```typescript
// Simuler une connexion 3G lente
await mcp__playwright__emulateNetworkConditions({
  offline: false,
  downloadThroughput: 500 * 1024,  // 500 KB/s
  uploadThroughput: 500 * 1024,
  latency: 100  // 100ms
});

// Naviguer et mesurer le temps de chargement
const startTime = Date.now();
await mcp__playwright__navigate({ url: 'http://localhost:3001/audits' });
await mcp__playwright__waitForSelector({ selector: '.ant-table' });
const loadTime = Date.now() - startTime;

console.log(`Temps de chargement avec 3G: ${loadTime}ms`);
```

### Test Responsive Mobile

```typescript
// Émuler un iPhone 12
await mcp__playwright__emulateDevice({ device: 'iPhone 12' });

// Naviguer vers Eole (portail client)
await mcp__playwright__navigate({ url: 'http://localhost:3002' });

// Capturer screenshot mobile
const mobileScreenshot = await mcp__playwright__screenshot({ fullPage: true });

// Vérifier que le menu hamburger est présent
await mcp__playwright__waitForSelector({ selector: '.mobile-menu-button' });
```

### Workflow Complet : Création d'Audit

```typescript
// 1. Connexion
await mcp__playwright__navigate({ url: 'http://localhost:3001/login' });
await mcp__playwright__fill({ selector: '[name="email"]', text: 'auditor@test.com' });
await mcp__playwright__fill({ selector: '[name="password"]', text: 'password' });
await mcp__playwright__click({ selector: 'button[type="submit"]' });
await mcp__playwright__waitForUrl({ url: '**/audits' });

// 2. Créer un nouvel audit
await mcp__playwright__click({ selector: 'button:has-text("Créer un audit")' });
await mcp__playwright__waitForSelector({ selector: 'form[name="audit-form"]' });

// 3. Remplir le formulaire
await mcp__playwright__fill({ selector: '[name="name"]', text: 'Test Audit Automatique' });
await mcp__playwright__select({ selector: '[name="customerId"]', value: '1' });
await mcp__playwright__select({ selector: '[name="siteId"]', value: '1' });

// 4. Screenshot du formulaire rempli
await mcp__playwright__screenshot({ fullPage: false });

// 5. Soumettre
await mcp__playwright__click({ selector: 'button[type="submit"]' });
await mcp__playwright__waitForUrl({ url: '**/audits/**' });

// 6. Vérifier la création
const content = await mcp__playwright__content();
// Vérifier que "Test Audit Automatique" apparaît
```

## Configuration Avancée

### Options de Navigateur

Pour passer des options au navigateur, modifiez `.mcp.json` :

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "-y",
        "@playwright/mcp@latest",
        "--browser=chromium",
        "--headless=true"
      ]
    }
  }
}
```

Options disponibles :
- `--browser` : `chromium`, `firefox`, `webkit`
- `--headless` : `true` ou `false` (mode sans interface)
- `--viewport-size` : `1920x1080` (dimensions)
- `--user-agent` : User agent personnalisé

## Bonnes Pratiques

### 1. Toujours Attendre les Éléments
```typescript
// ❌ Mauvais : Cliquer directement
await mcp__playwright__click({ selector: 'button.submit' });

// ✅ Bon : Attendre que l'élément soit présent
await mcp__playwright__waitForSelector({ selector: 'button.submit' });
await mcp__playwright__click({ selector: 'button.submit' });
```

### 2. Utiliser des Sélecteurs Robustes
```typescript
// ❌ Éviter : Sélecteurs fragiles
'div.css-abc123 > span:nth-child(2)'

// ✅ Préférer : Sélecteurs sémantiques
'[data-testid="submit-button"]'
'button:has-text("Soumettre")'
'[aria-label="Fermer"]'
```

### 3. Screenshots pour le Debug
```typescript
// Capturer des screenshots à chaque étape importante
await mcp__playwright__screenshot({ fullPage: false });
```

### 4. Nettoyer Après les Tests
```typescript
// Fermer le navigateur à la fin
await mcp__playwright__close();
```

## Intégration avec l'Agent Simulation

L'agent simulation (`.claude/agents/simulation.md`) utilise automatiquement ces outils pour :

1. **Tests E2E Automatisés** : Workflows complets de création d'audit
2. **Tests Multi-Rôles** : Basculer entre Auditor, Approbator, Customer
3. **Validation Visuelle** : Screenshots avant/après
4. **Tests de Performance** : Mesure avec network throttling
5. **Tests Responsive** : Validation mobile/desktop

## Dépannage

### Le serveur MCP ne démarre pas
```bash
# Vérifier que npx fonctionne
npx -y @playwright/mcp@latest --help

# Vérifier les logs de Claude Code
# Les logs MCP apparaissent dans la sortie de démarrage
```

### Les outils MCP n'apparaissent pas
1. Vérifier que `.mcp.json` existe à la racine du projet
2. Redémarrer Claude Code complètement
3. Vérifier dans la liste des outils disponibles

### Erreur de timeout
```typescript
// Augmenter le timeout pour les opérations lentes
await mcp__playwright__waitForSelector({
  selector: '.slow-element',
  timeout: 30000  // 30 secondes
});
```

## Ressources

- [Documentation Playwright MCP](https://github.com/microsoft/playwright-mcp)
- [Documentation Playwright](https://playwright.dev/)
- [Sélecteurs Playwright](https://playwright.dev/docs/selectors)
- [Agent Simulation APIACC](./agents/simulation.md)

## Version
Guide v1.0 - Configuration initiale Playwright MCP pour APIACC
