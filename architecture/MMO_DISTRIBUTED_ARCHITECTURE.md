# Architecture Serveur – MMO Distribué (v0)

> Document d’architecture de référence  
> Version : v0  
> Objectif : servir de base durable pour l’implémentation, l’évolution et la discussion de l’architecture serveur d’un MMO massivement multijoueur.

---

# 0. Contexte & Vision

Ce document décrit une architecture serveur conçue pour :
- supporter **un très grand nombre de joueurs dans un même combat**
- permettre un **scaling horizontal réel**
- rester **résiliente face aux pannes**
- rester **déboguable et observable**
- éviter les plafonds structurels observés dans les architectures “grid / lockstep”

L’architecture privilégie :
- la clarté des responsabilités
- la tolérance à l’asynchronisme
- la dégradation contrôlée plutôt que le blocage global

---

# 1. Principes Fondamentaux

1. **Un seul writer par entité**
2. **Pas de lockstep global**
3. **Pas de dépendance réseau synchrone dans le tick**
4. **Cache toujours jetable**
5. **Autorité clairement définie**
6. **Observabilité dès le départ**
7. **Résilience par design, pas par rattrapage**

---

# 2. Vue d’Ensemble de l’Architecture

[ Clients ]
|
[ Gateways ]
|
[ Compute Nodes ]
|
[ Coordinator + Vice ]


Chaque composant a un rôle strictement limité.

---

# 3. Clients

## Rôle
- Interface joueur
- Envoi des inputs
- Réception de l’état visible du monde

## Règles
- Ne connaît pas l’ownership
- Ne connaît pas la topologie serveur
- Ne gère aucune subscription
- Ne participe pas à la résilience
- Parle uniquement à **une gateway**

Le client est volontairement simple et ignorant de l’architecture interne.

---

# 4. Gateways

## Rôle
Les gateways sont des **proxies intelligents et stateful** :
- maintiennent les connexions client
- routent les inputs vers les bons compute nodes
- agrègent et filtrent les deltas serveur
- gèrent les subscriptions côté client
- mettent en cache l’état client-facing

## Propriétés
- Sessions **sticky** (client ↔ gateway fixe)
- Cache local, borné, jetable
- Aucun calcul gameplay
- Aucune autorité

## Résilience
- Gateway KO → reconnexion client
- Migration client prévue ultérieurement (v2)

---

# 5. Compute Nodes (Nodes de Calcul)

## Rôle
- Exécuter **toute la simulation gameplay**
- Chaque node est identique (même binaire)

Les compute nodes sont le cœur du moteur de simulation.

---

# 6. Ownership (Autorité par Entité)

## Principe
Chaque entité (joueur, PNJ, projectile, etc.) :
- a **exactement un owner**
- peut avoir **zéro ou plusieurs readers (ghosts)**

> Un seul writer, toujours.

## Bénéfices
- pas de write concurrent
- pas de lock distribué
- distribution naturelle de la charge
- recovery localisé en cas de panne

---

# 7. Dépendances & Graphe Gameplay

Chaque compute node travaille uniquement sur :
- les entités dont il est owner
- les **ghosts nécessaires** au gameplay local :
  - targets lockées
  - voisins proches
  - membres de fleet/squad
  - entités en interaction active

Le graphe est :
- dynamique
- clairsemé
- borné

---

# 8. Subscriptions Inter-Nodes

## Principe
Les nodes s’abonnent explicitement aux données dont ils ont besoin.

## Règles
- Subscribe explicite
- Push initial + deltas
- TTL + hysteresis
- Aucun broadcast
- Aucun pull synchrone dans le tick

Les subscriptions sont **hors boucle chaude**.

---

# 9. Queues (Fondation du Système)

## Principe Clé
> Toutes les queues vivent dans les compute nodes.  
> Le réseau est un transport, jamais une queue logique.

## Queues par Node
- **InputQueue** : inputs joueurs / IA
- **IntentQueue** : intentions gameplay (locales ou réseau)
- **SimulationQueue** : jobs internes
- **OutputQueue** : deltas produits

---

# 10. Tick Serveur

## Cadence
- Tick cible : **10 Hz** (100 ms)

## Déroulé
1.Drain InputQueue

2.Générer des intents

3.Simulation locale

4.Consommer IntentQueue

5.Produire des deltas

6.Flush OutputQueue


## Contraintes
- aucune attente réseau
- aucune découverte dynamique
- tout est préparé hors tick

---

# 11. Coordinator (Dédié)

## Rôle
- gérer la table `entity → owner`
- surveiller les nodes
- équilibrer la charge
- décider des migrations
- maintenir l’epoch global

## Design
- 1 coordinateur leader
- 1 vice-coordinateur (hot standby)
- réplication leader → vice

Le coordinateur n’exécute **aucun gameplay**.

---

# 12. Epoch (Régime de Vérité)

## Définition
Un **epoch** représente le régime de vérité du cluster.

## Règles
- un seul epoch actif
- tout message de contrôle contient un epoch
- message.epoch < currentEpoch → ignoré
- epoch++ lors d’un failover coordinateur

## Objectif
- éviter le split-brain
- invalider automatiquement les ordres anciens
- simplifier la gestion des pannes

---

# 13. Gestion des Pannes

## Compute Node KO
- détection par heartbeat
- ownership invalidé
- redistribution par le coordinateur
- recovery via ghosts + snapshot

Impact :
- local
- ≤ 1–3 ticks

## Coordinateur KO
- le vice devient leader
- epoch++

## Gateway KO
- reconnexion client
- cache perdu (acceptable)

---

# 14. Charge & Headroom (N+1)

## Principe
Pour survivre à la perte d’un node :
Charge cible ≤ N / (N + 1)

Exemples :
- 5 nodes → ~80 %
- 8 nodes → ~87 %

## Types de Charge
- charge dure (non compressible)
- charge molle (LOD, réplication)
- charge opportuniste (prefetch, confort)

---

# 15. Réseau & Protocoles

## Client ↔ Gateway
- TCP + TLS
- connexions longues

## Inter-serveur
- Control plane : TCP
- Data plane : TCP (v1), UDP / QUIC (v2)

Réseau interne :
- isolé
- authentification légère
- validation stricte des messages

---

# 16. Admin & Debug

## Client Admin
- parle au coordinateur
- commandes explicites
- traçables
- usage non-prod prioritaire

## Epoch Admin
- `epoch = INT_MAX`
- supplantation de toute autorité
- extrêmement contrôlé en prod

---

# 17. Observabilité (Indispensable)

## Identité partout
- EntityID
- TickID
- Epoch
- NodeID

## Outils attendus
- timeline par entité
- graphe d’ownership
- visualisation des migrations
- métriques de queues
- mode debug ciblé

---

# 18. Plan de Livraison

## v1
- mono-node
- gateway + compute node
- 3–5 features gameplay
- architecture déjà “distribué-ready”

## v2
- coordinateur + vice
- 2 compute nodes
- premiers vrais combats multi-joueurs
- observabilité renforcée

## v3+
- scaling massif
- LOD serveur
- failover quasi invisible

---

# 19. Lexique

## Lockstep
Mode où tous les participants doivent attendre tout le monde pour avancer.  
Évité ici car non scalable.

## Ownership
Responsabilité unique d’un serveur sur une entité.

## Stateful
Composant conservant un état en mémoire (nodes, gateways).

## Stateless
Composant sans mémoire du passé (client autant que possible).

## Sticky
Session collante : un client reste sur la même gateway.

## Subscribe / Unsubscribe
Mécanisme d’abonnement aux mises à jour.

## Ghost
Copie partielle et non autoritaire d’une entité.

## Tick
Pas de temps logique serveur.

## Epoch
Numéro représentant le régime de vérité du cluster.

## Failover
Capacité à continuer malgré une panne.

---

# Conclusion

> Cette architecture est conçue pour **tenir sous la pression**,  
> **évoluer sans refonte**,  
> et **éviter les plafonds structurels**.

Elle privilégie :
- la clarté
- la résilience
- la scalabilité
- l’observabilité

Ce document est la **base de référence** pour toute évolution future.
