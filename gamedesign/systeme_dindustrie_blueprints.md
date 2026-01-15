# MMO Spatial – Système d’Industrie & Blueprints

## Objectif du document
Ce document décrit **le gameplay d’industrie**, le **système de blueprints évolutifs**, les **arbres de bonus**, et le **modèle de données** associé.

Il sert de **référence de design** et de **base d’implémentation technique**.

---

## Vision générale

L’industrie n’est pas un simple système de fabrication, mais un **pilier de gameplay à part entière** :
- Les joueurs conçoivent des *blueprints uniques*
- Les blueprints évoluent avec l’usage
- Les chaînes de montage influencent la qualité et le rendement
- Chaque vaisseau produit est traçable
- La destruction alimente l’économie

---

## Concepts fondamentaux

### Blueprint
Un blueprint est un **objet vivant** qui représente un savoir-faire industriel.

Il possède :
- une expérience (XP)
- un niveau
- un arbre de bonus alloué
- des traits et défauts
- une valeur économique propre

### Séparation des responsabilités
- **Ingénieur** : conçoit et améliore les blueprints
- **Blueprint** : porte la spécialisation
- **Chaîne de montage** : produit en masse

---

## Progression & XP

### Triple progression

| Niveau | Rôle |
|------|------|
| Ingénieur | Débloque types de bonus et efficacité globale |
| Blueprint | Spécialisation et identité unique |
| Chaîne de montage | Rendement, stabilité, défauts |

---

## Blueprints

### BlueprintTemplate (statique)
Représente le modèle de base commun.

- Type (Hull, Weapon, Engine…)
- Classe (Frigate, Cruiser…)
- Stats de base
- Arbre de talents associé

### Blueprint (instance unique)

- Référence un BlueprintTemplate
- Accumule de l’XP
- Dispose de points de spécialisation
- Alloue des nœuds dans un arbre
- Peut être copié, vendu, volé ou détruit

---

## Exemple : Châssis de frégate de combat

### Tronc commun

Branches disponibles :
- Mobilité & Cinématique
- Défense & Survivabilité
- Structure & Production
- Systèmes & Modularité

Chaque branche impose des **choix irréversibles** et des **trade-offs**.

---

## Spécialisation : Frégate anti-frégate

### Philosophie
- Supériorité en duel
- Alpha strike
- Mobilité élevée
- Faible endurance prolongée

### Branches
- Tracking & précision
- Alpha strike
- Disruption & contrôle

La spécialisation est **irréversible**.

---

## Chaînes de montage

Les chaînes de montage sont des entités configurables :
- Capacités supportées
- Modificateurs de production
- Stabilité
- Risque de défauts

Certaines spécialisations de blueprint exigent des capacités industrielles spécifiques.

---

## Unicité & héritage

Les blueprints peuvent :
- être copiés avec perte
- être rétro-ingéniérés
- être pillés
- évoluer différemment selon l’usage

Deux blueprints identiques à l’origine peuvent diverger fortement.

---

## Modèle de données

### Player
```json
{
  "playerId": "UUID",
  "name": "string",
  "corporationId": "UUID",
  "engineerProfileId": "UUID"
}
```

### EngineerProfile
```json
{
  "engineerId": "UUID",
  "level": 12,
  "xp": 125000,
  "specializations": ["Hull"],
  "passiveBonuses": []
}
```

### BlueprintTemplate
```json
{
  "templateId": "UUID",
  "type": "Hull",
  "class": "Frigate",
  "role": "Combat",
  "baseStats": {},
  "skillTreeId": "UUID"
}
```

### SkillTree / SkillNode
```json
{
  "nodeId": "UUID",
  "name": "Allègement structurel",
  "cost": 1,
  "effects": []
}
```

### Blueprint
```json
{
  "blueprintId": "UUID",
  "templateId": "UUID",
  "ownerId": "UUID",
  "xp": 42000,
  "level": 8,
  "allocatedNodes": [],
  "traits": [],
  "defects": [],
  "version": 2
}
```

### AssemblyLine
```json
{
  "lineId": "UUID",
  "capabilities": {},
  "modifiers": [],
  "stability": 0.91
}
```

### ProductionJob
```json
{
  "jobId": "UUID",
  "blueprintId": "UUID",
  "quantity": 5,
  "materialCost": {}
}
```

### ProducedItem
```json
{
  "itemId": "UUID",
  "blueprintSnapshot": {},
  "finalStats": {},
  "history": [],
  "status": "Active"
}
```

---

## Calcul des statistiques

Les statistiques finales ne sont jamais stockées sur le blueprint.

Formule :

```
FinalStat =
  BaseTemplateStat
+ SkillNodeEffects
+ EngineerBonuses
+ AssemblyLineModifiers
+ RandomVariance
```

---

## Architecture recommandée

- Blueprints & Items : stockage document (JSON)
- Templates & arbres : stockage statique / relationnel
- Événements (destruction) : event store

Pattern recommandé : CQRS léger.

---

## Principes clés

- Pas de blueprint parfait
- Perte = moteur économique
- Spécialisation > optimisation
- L’industrie raconte des histoires

---

## Prochaines extensions possibles

- Défauts émergents
- Piratage industriel
- Blueprints légendaires
- IA de marché
- Méta-économie inter-systèmes

---

*Fin du document*