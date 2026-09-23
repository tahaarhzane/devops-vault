---
titre: Architecture Kubernetes
domaine: Kubernetes
tags: [kubernetes, architecture, control-plane, node]
derniere_revision: 2026-09-23
---

# Architecture Kubernetes

## Contexte

Un cluster Kubernetes est composé de deux types de machines : les nœuds du **control plane**
(qui pilotent le cluster) et les **worker nodes** (qui exécutent les charges de travail).
Comprendre ce découpage est la base pour diagnostiquer n'importe quel problème (pourquoi un
pod ne démarre pas, pourquoi le scheduling échoue, pourquoi l'API ne répond plus, etc.).

## Notes

### 🗺️ Vue d'ensemble

![Vue d'ensemble : control plane et worker nodes](assets/architecture-overview.svg)
*Le control plane (détail dans le schéma suivant) pilote deux worker nodes : kubelet reçoit les instructions, démarre les conteneurs via le runtime, pendant que kube-proxy applique les règles réseau vers les pods.*

### ⚙️ Composants du control plane

![Détail des interactions du control plane](assets/architecture-control-plane-detail.svg)
*Aucun composant ne parle directement à un autre : le scheduler assigne les pods en attente, le controller-manager réconcilie l'état, le cloud-controller-manager provisionne les ressources cloud — tous via l'apiserver, seul à parler à etcd.*

| Composant | Rôle |
|---|---|
| **kube-apiserver** | point d'entrée unique de l'API Kubernetes (REST). Toutes les interactions (kubectl, controllers, kubelet) passent par lui. Stateless, scalable horizontalement. |
| **etcd** | base de données clé-valeur distribuée, stocke l'état complet du cluster (source de vérité). |
| **kube-scheduler** | assigne les pods non planifiés à un nœud, selon les ressources disponibles, les contraintes (affinity, taints/tolerations, resource requests). |
| **kube-controller-manager** | fait tourner les boucles de contrôle (control loops) qui rapprochent l'état réel de l'état désiré (ex. controller de Deployment, de Node, de Job). |
| **cloud-controller-manager** | fait l'interface avec le cloud provider (Azure, AWS...) pour les LoadBalancer, volumes, et labels de nœuds spécifiques au cloud. |

> [!CAUTION]
> Un etcd corrompu ou perdu = cluster perdu (plus aucun état, plus aucune ressource
> retrouvable) → des sauvegardes etcd régulières et testées sont non négociables.

### 🖥️ Composants des worker nodes

- **kubelet** : agent sur chaque nœud, communique avec l'apiserver, s'assure que les
  conteneurs décrits dans les PodSpecs tournent et sont en bonne santé (probes).
- **kube-proxy** : maintient les règles réseau sur le nœud (iptables/IPVS) pour permettre
  la communication vers les Services — voir [services-reseau.md](services-reseau.md).
- **Container runtime** : exécute réellement les conteneurs (containerd, CRI-O), via
  l'interface CRI (Container Runtime Interface).

### ✅ Principe clé : reconciliation loop

> [!IMPORTANT]
> Tout Kubernetes fonctionne sur un modèle déclaratif : on décrit un **état désiré** (manifest
> YAML appliqué via l'API), et des controllers comparent en continu cet état désiré à l'état
> réel observé, puis appliquent les actions nécessaires pour converger. C'est ce mécanisme qui
> explique la résilience de K8s (un pod tué est automatiquement recréé par le controller du
> Deployment, par exemple).

## Références

- [Kubernetes Components (doc officielle)](https://kubernetes.io/docs/concepts/overview/components/)
- [etcd documentation](https://etcd.io/docs/)
