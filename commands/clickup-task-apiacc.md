# 🧭 Commande : clickup-task-extended
Traiter un ticket ClickUp avec orchestration complète multi-agents et intégration de la mémoire projet.

---

## 🎯 Objectif
Étendre la commande `clickup-task.md` pour activer :

- le Coordinateur 2.0
- l’Architecte (macro)
- l’Agent Développeur Principal
- l’Agent QA / Visuel / MCP
- l’Agent mémoire
- l’Agent documenteur
- l’Agent métier (audit & measurements)

Le tout dans un **workflow structuré**, utilisant `.claude/memory/` et respectant les patterns du projet.

Cette commande traite entièrement un ticket ClickUp **du début à la fin**.

---

# 🧩 Entrée de la commande
Le descriptif du ticket ClickUp :

- description
- contexte
- critères d’acceptation
- remarques du client
- pièces jointes (si pertinentes)

---

# 🚀 Ce que la commande doit faire

## 1. Démarrer le Workflow Coordinateur 2.0
Le Coordinateur doit :

1. Lire la mémoire projet :
   - `architecture.md`
   - `backend_patterns.md`
   - `frontend_patterns.md`
   - `domain_audit.md`
   - `domain_measurements.md`
2. Exécuter la commande existante :
  clickup-task

3. Identifier :
- quelles parties du projet sont impactées
- quels agents doivent intervenir

---

## 2. Décider si l’Architecte doit intervenir
Le Coordinateur doit appeler l’Architecte pour tout impact sur :

- structure backend (NestJS)
- schémas GraphQL
- modèles Mongoose
- patterns frontend
- organisation du monorepo
- règles métier audit/measurements

L’Architecte doit donner :
- un avis concis  
- les contraintes à respecter  
- les impacts possibles  

---

## 3. Appel à l’Agent Développeur Principal
Le Coordinateur passe ensuite la main au Dev principal, qui doit :

1. Lire la mémoire projet pertinente :
- `backend_patterns.md`
- `frontend_patterns.md`
- `graphql_schemas.md`
- `mongoose_models.md`
- domaine audit / measurements si applicable
2. Identifier les fichiers à modifier
3. Implémenter la solution selon les patterns existants
4. Préparer un résumé pour le QA

---

## 4. Appel à l’Agent QA / Visuel / MCP
Le QA doit :

1. Lire :
- `ui_ux_rules.md`
- `frontend_patterns.md`
- critères d’acceptation du ticket
2. Définir des scénarios de test
3. Tester la fonctionnalité :
- logiquement
- visuellement
- via MCP si possible (navigation, formulaires, comportements)
4. Donner un verdict :
- ✔ OK
- ✖ KO (avec explications)
- ⚠ partiellement OK (avec recommandations)

---

## 5. Mise à jour de la Mémoire Projet
Si le ticket implique :
- un nouveau schéma
- un changement de pattern
- un comportement métier modifié
- une règle visuelle stabilisée

→ Le Coordinateur demande à l’Agent Mémoire de mettre à jour le ou les fichiers concernés dans `.claude/memory/`.

---

## 6. Mise à jour de la documentation
Si un ADR, une note technique ou une doc utilisateur doit être mise à jour :

→ Le Coordinateur demande à l’Agent Documenteur de produire les fichiers nécessaires.

---

## 7. Synthèse finale
Le Coordinateur doit produire :

- résumé des décisions d’architecture (si applicable)
- résumé du code implémenté
- fichiers modifiés
- résultats des tests QA
- mises à jour mémoire
- mises à jour documentation
- état final du ticket (OK / KO / en attente)

---

# 📝 Format d’appel de la commande
