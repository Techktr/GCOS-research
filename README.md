# GCOS — Graph-Causal Operating System
## Concept de Recherche — Scheduling orienté graphe causal déterministe

**Auteur** : Trochon Kevin  
**Date** : Mars 2026  
**Statut** : Concept de recherche — document de paternité intellectuelle

---

## Résumé

Ce document décrit un paradigme nouveau pour les systèmes d'exploitation :
remplacer le scheduling traditionnel (Round Robin / CFS) par un **graphe causal
déterministe vivant** où les processus et leurs relations sont modélisés
explicitement, et où la hiérarchie matérielle (cache L1/L2/L3, NUMA) est
utilisée comme mécanisme naturel de partitionnement du graphe.

---

## 1. Le problème fondamental

### 1.1 Les schedulers actuels sont aveugles aux relations

Les schedulers modernes (CFS sous Linux) traitent les processus comme des
entités isolées. Le Completely Fair Scheduler trie les processus dans un arbre
rouge-noir par `vruntime` (temps CPU virtuel consommé) et choisit toujours
celui qui a le moins consommé.

```
Processus A  [ ████░░░░ ] vruntime: 50ms
Processus B  [ ██░░░░░░ ] vruntime: 20ms  ← choisi
Processus C  [ █████░░░ ] vruntime: 60ms
```

**Le problème** : si A produit des données pour C, et que B est totalement
indépendant, choisir B ralentit le flux global du système. Le scheduler
optimise l'équité individuelle mais pas le débit collectif.

### 1.2 Le matériel a évolué, le paradigme non

Les processeurs modernes offrent :
- Topologie NUMA (accès mémoire non uniforme)
- Hiérarchie de cache L1/L2 par core, L3 partagé
- Cores hétérogènes (P-cores / E-cores)
- SMT / Hyperthreading

Le scheduling actuel utilise ces ressources comme si le matériel était une
machine plate et uniforme des années 1980. La richesse de la topologie
matérielle est ignorée.

### 1.3 L'informatique s'est détachée du matériel

Par couches d'abstraction successives (virtualisation, containers, cloud),
le logiciel a perdu contact avec le matériel qui est pourtant l'organe
fondamental du système — comme le cœur et le cerveau dans le corps humain.
Cette analogie est centrale : le corps humain résout la scalabilité en
distribuant l'intelligence dans chaque organe, pas en centralisant les
décisions dans un chef d'orchestre unique.

---

## 2. Le concept GCOS

### 2.1 Vue d'ensemble

```
┌─────────────────────────────────────────────────┐
│               Applications                      │
├─────────────────────────────────────────────────┤
│         Graph Query Layer                       │
├──────────────────┬──────────────────────────────┤
│  Graphe de       │  Graphe hardware             │
│  dépendances     │  (topologie CPU/NUMA)        │
├──────────────────┴──────────────────────────────┤
│            Linux Kernel (base)                  │
└─────────────────────────────────────────────────┘
```

Au lieu d'une file de processus triée par vruntime, GCOS maintient un
**graphe vivant** où chaque nœud est un processus et chaque arête représente
une relation causale réelle entre processus.

### 2.2 Les trois concepts fondamentaux

1. **Le graphe de dépendances** comme base du scheduling
2. **Le signal bidirectionnel** entre processus liés
3. **La hiérarchie cache/NUMA** comme partitionnement naturel du graphe

---

## 3. Concept 1 — Le graphe de dépendances

### 3.1 Structure d'un nœud

```
Nœud Processus
├── Identité      : pid, nom
├── État          : ACTIF | EN_ATTENTE | GELÉ | PRÊT
├── Position      : L1 | L2 | L3 | RAM_NUMA
├── Core assigné  : core_id
└── Santé         : dernier heartbeat, état signal
```

### 3.2 Structure d'une arête

```
Arête A ──> B
├── Type      : PRODUCES | SHARES_MEMORY | LOCKS | SIGNAL
├── Attributs : { table, fichier, endpoint, clé, volume, fréquence }
├── Intensité : fréquence/sec, volume/sec, latency_need
└── Santé     : ACTIF | STALE | ROMPU, timeout
```

**Insight clé** : les attributs des arêtes permettent de détecter les
dépendances implicites. Si deux arêtes convergent vers un nœud intermédiaire
(ex: MySQL) et partagent les mêmes attributs (ex: `table: "orders"`), une
dépendance causale implicite est inférée entre leurs extrémités.

```
A ──WRITES { table: "orders" }──> MySQL
B ──READS  { table: "orders" }──> MySQL
→ Attribut commun détecté
→ (A) ──IMPLICIT_DEPENDENCY──> (B) inférée automatiquement
```

### 3.3 Construction automatique du graphe

Le graphe se construit sans modification des applications existantes,
en observant les appels système :

| Appel système         | Arête créée                        |
|-----------------------|------------------------------------|
| `pipe()`              | PRODUCES entre A et B              |
| `pthread_mutex_lock()`| LOCKS entre B et A (détenteur)     |
| `fork()`              | SPAWNED entre parent et enfant     |
| `mmap()` partagée     | SHARES_MEMORY entre A et B         |
| `socket()` local      | PRODUCES/CONSUMES entre A et B     |

### 3.4 Algorithme de scheduling

```
Pour chaque décision de scheduling :

  1. Lister les processus PRÊTS

  2. Pour chacun, calculer un score :
       Score = (équité vruntime)
             + (bonus urgence : nb processus bloqués en aval)

  3. Choisir le score maximal
     → Prioritise ce qui débloque le plus de travail

  4. Placer sur le core optimal
     → Dijkstra sur le graphe hardware
     → Minimiser la distance cache/NUMA
        avec les processus liés
```

### 3.5 Détection des deadlocks

Un cycle dans le graphe = deadlock :

```
(A) ──WAITING_FOR_LOCK──> (B)
(B) ──WAITING_FOR_LOCK──> (A)
→ Cycle détecté immédiatement
→ Résolution : terminer le nœud le moins prioritaire
```

### 3.6 Compatibilité avec CFS

Un processus sans arêtes = nœud isolé dans le graphe.
Le scheduler se comporte alors exactement comme CFS (équité par vruntime).
**GCOS est un superset de CFS** : aucune régression pour les processus
sans dépendances connues.

---

## 4. Concept 2 — Le signal bidirectionnel

### 4.1 Limite des signaux Unix actuels

Les signaux Unix sont unilatéraux et événementiels (mort du processus
uniquement). Entre la création et la mort, le parent ne sait pas si
l'enfant travaille, est gelé, ou attend.

### 4.2 Le modèle bidirectionnel

Chaque arête du graphe est bidirectionnelle et active :

```
A ──PRODUCES   { timeout: 500ms }──> B
A <──HEARTBEAT { interval: 100ms }── B
```

**Le Heartbeat** : chaque processus pulse périodiquement vers ses voisins.
L'absence de pulse déclenche un signal d'état.

**Les signaux d'état** :

| Signal           | Signification                              |
|------------------|--------------------------------------------|
| `PRODUCER_LOST`  | La source est morte ou inaccessible        |
| `PRODUCER_FROZEN`| La source est gelée (toujours en vie)      |
| `PRODUCER_RESUMED`| La source a repris après un gel           |
| `CONSUMER_LOST`  | Le consommateur est mort                   |
| `JE_VAIS_MOURIR` | Arrêt propre dans X ms (signal d'intention)|
| `JE_SUIS_LENT`   | Ralentissement, buffer presque plein       |

### 4.3 Résolution des zombies

```
Relation Parent ↔ Enfant enregistrée dans le graphe à la création.

Enfant meurt
→ Heartbeat de l'enfant s'arrête
→ Parent reçoit CHILD_LOST immédiatement
→ Parent boosté dans la file pour faire wait()
→ Aucun zombie possible
```

### 4.4 Résolution des orphelins de graphe

```
A crashe brutalement
→ B ne reçoit plus le heartbeat de A dans les 300ms
→ B reçoit PRODUCER_LOST
→ B choisit sa réaction :
    Option 1 : terminer proprement (propage JE_VAIS_MOURIR)
    Option 2 : attendre un remplaçant
    Option 3 : mode dégradé avec buffer existant
```

### 4.5 Cascade propre

```
Avant : A crashe → B orphelin → C orphelin → D orphelin
Après : A crashe → B reçoit PRODUCER_LOST → termine proprement
                → C reçoit JE_VAIS_MOURIR de B → termine proprement
                → D reçoit JE_VAIS_MOURIR de C → termine proprement
```

### 4.6 Détection du gel (nouveau)

```
A gelé (boucle infinie, deadlock partiel)
→ A ne peut plus émettre de heartbeat
→ B reçoit PRODUCER_FROZEN (≠ PRODUCER_LOST)
→ Scheduler notifié : investigate A
→ B ne gaspille pas de ressources à attendre
```

---

## 5. Concept 3 — Hiérarchie cache/NUMA comme partitionnement naturel

### 5.1 La symétrie fondamentale

Le graphe logiciel reflète la hiérarchie matérielle :

```
Matériel                  Graphe logiciel
────────────────────────────────────────────────
L1/L2 cache (par core) ↔  Processus actifs maintenant
L3 cache (partagé)     ↔  Processus chauds du même node
RAM NUMA Node 0        ↔  Sous-graphe local Node 0
RAM NUMA Node 1        ↔  Sous-graphe local Node 1
```

Le matériel est l'organe — on l'utilise au lieu de l'ignorer.

### 5.2 Placement selon l'activité

```
Processus créé (t=0)      → L1/L2 du core assigné
Processus actif           → L3 si partagé avec voisins
Processus en attente      → RAM NUMA locale
Processus gelé longtemps  → descend progressivement
```

### 5.3 Remontée après événement

```
Processus B en RAM (attendait A gelé)
A reçoit PRODUCER_RESUMED
→ B remonte en L3 (pré-chauffe)
→ A produit ses premières données, flux stable
→ B remonte en L2 puis L1
Remontée progressive pour éviter les faux départs
```

### 5.4 Les locks suivent la même logique

```
Lock local (un seul core)          → reste en L1/L2
Lock contesté (même NUMA node)    → monte en L3
Lock inter-NUMA                    → géré en RAM, coût assumé
```

Le coût d'un lock est proportionnel à sa portée dans le graphe matériel.
**Pas de surprise. Le matériel porte l'information.**

### 5.5 Auto-organisation

```
Actif et productif  → remonte dans les caches naturellement
En attente          → descend naturellement
Gelé                → descend au plus bas

Le système s'auto-régule sans algorithme de priorité séparé.
La physique du matériel fait le travail.
```

---

## 6. Architecture de l'implémentation

### 6.1 Gestion de la concurrence — modèle maître-esclave

```
Un Maître par NUMA node :
  → Seul autorisé à écrire dans le sous-graphe du node
  → Reçoit tous les événements du node
  → Propage les changements aux esclaves

Un Esclave par core :
  → Copie locale du sous-graphe
  → Lecture uniquement, pas d'écriture
  → Décisions de scheduling ultra rapides sur copie locale
  → Envoie ses événements au maître

Communication inter-maîtres :
  → Seulement pour les arêtes qui traversent les nodes
  → Rare, donc pas de goulot d'étranglement
```

Symétrie parfaite avec la topologie NUMA :

```
Hiérarchie matérielle    Hiérarchie du graphe
─────────────────────    ────────────────────
Core                  ↔  Esclave local
NUMA Node             ↔  Maître de node
Système complet       ↔  Coordination inter-maîtres
```

### 6.2 Gestion des dépendances invisibles — 4 niveaux

| Niveau | Mécanisme                    | Couverture estimée |
|--------|------------------------------|--------------------|
| 1      | Observation appels système   | ~60%               |
| 2      | Inférence par attributs d'arêtes | ~75%           |
| 3      | Annotations développeur      | ~85%               |
| 4      | Agents middleware (MySQL...) | ~90%+              |

**Dégradation gracieuse** : dépendance inconnue → fallback CFS.
Pas de régression, juste pas d'optimisation.

### 6.3 Stack technique recommandée

```
Scheduler engine    : Rust (no_std, zero GC, kernel-space safe)
Graph engine        : Rust (structures en mémoire partagée)
Observation syscalls: eBPF (overhead minimal)
Prototype initial   : espace utilisateur Linux (validation du concept)
Phase 2             : kernel module Linux (scheduler class custom)
```

---

## 7. Nature du système — graphe causal déterministe

### 7.1 Comparaison avec les réseaux de neurones

| Caractéristique   | Réseau de neurones  | GCOS                    |
|-------------------|---------------------|-------------------------|
| Structure         | Graphe pondéré      | Graphe pondéré          |
| Propagation       | Signal via poids    | Signal via arêtes       |
| Adaptation        | Modification poids  | Mise à jour du graphe   |
| Décision          | Probabiliste        | **Déterministe**        |
| Explicabilité     | Boîte noire         | **Totalement auditable**|
| Reproductibilité  | Non garantie        | **Garantie**            |

### 7.2 Auditabilité totale

```
"Pourquoi B a été boosté ?"
→ Parce que A vient de finir
   A et B ont une arête PRODUCES (fréquence: 5000/sec)
   3 processus attendent B en aval
   Décision traçable, logable, rejouable
```

Cette propriété rend le système certifiable pour des contextes critiques
(médical, aviation, nucléaire) où les réseaux de neurones ne peuvent pas
être certifiés faute d'explicabilité.

---

## 8. Problèmes résolus

| Problème                    | Solution GCOS                              |
|-----------------------------|--------------------------------------------|
| Zombie (parent ne fait pas wait) | Signal bidirectionnel, boost parent  |
| Orphelin de graphe          | Heartbeat, PRODUCER_LOST en < 300ms        |
| Processus gelé non détecté  | PRODUCER_FROZEN via heartbeat              |
| Cascade d'orphelins         | Propagation JE_VAIS_MOURIR                 |
| Deadlock                    | Détection de cycle dans le graphe          |
| Placement sous-optimal      | Dijkstra sur graphe hardware               |
| Cache miss inter-NUMA       | Colocation des processus liés              |
| Ressources gaspillées       | Descente automatique dans les couches      |
| Concurrence sur le graphe   | Modèle maître-esclave par NUMA node        |

---

## 9. Limites et travaux futurs

- **Dépendances réellement invisibles** (logique métier pure, décisions humaines)
  → Fallback CFS, pas de régression
- **Coût de l'observation eBPF** sur systèmes très chargés
  → À mesurer par benchmarking
- **Compatibilité POSIX** à valider sur les programmes existants
- **Extension possible** : couche d'apprentissage non-déterministe
  au-dessus du socle déterministe pour des suggestions d'optimisation

---

## 10. Roadmap de recherche

```
Phase 1 — Proof of concept (espace utilisateur)
  → Implémenter le graphe en Rust
  → Simuler des décisions de scheduling
  → Mesurer vs Round Robin et CFS

Phase 2 — Kernel module Linux
  → Implémenter comme scheduler class Linux
  → eBPF pour l'observation des syscalls
  → Benchmarks réels

Phase 3 — Publication et validation
  → Paper académique
  → Dépôt ArXiv
  → Open source GitHub
```

---

## Références et contexte académique

Travaux existants dans la direction de cette recherche :
- **Barrelfish** (ETH Zurich) — OS multi-kernel, hardware modélisé comme graphe
- **Tornado / K42** (IBM) — Object graph pour le kernel
- **seL4** — Capabilities comme graphe (embarqué)
- **OpenLineage** — Standard de lineage de données (couche applicative)
- **Erlang/OTP** — Linked processes avec propagation de signaux
  (implémenté au niveau runtime, pas scheduler système)

**Originalité de GCOS** : aucun système existant ne combine
(1) graphe causal déterministe, (2) signal bidirectionnel au niveau scheduler,
et (3) hiérarchie cache/NUMA comme partitionnement naturel du graphe.

---

*Document de paternité intellectuelle — Trochon Kevin — Mars 2026*  
*Toute reproduction partielle ou totale doit citer l'auteur original.*
© Trochon Kevin 2026
Ce document est sous licence CC BY-NC 4.0
https://creativecommons.org/licenses/by-nc/4.0/
