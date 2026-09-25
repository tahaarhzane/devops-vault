---
titre: Deployment & ReplicaSet
domaine: Kubernetes
tags: [kubernetes, deployment, replicaset, rolling-update]
derniere_revision: 2026-09-23
---

# Deployment & ReplicaSet

## Contexte

Le Deployment est le controller le plus utilisé pour des applications **stateless**. Il
gère indirectement les Pods via un ReplicaSet intermédiaire, ce qui permet les mises à
jour progressives (rolling updates) et les rollbacks.

## Notes

### ⚙️ Hiérarchie

![Hiérarchie Deployment, ReplicaSet, Pods](assets/deployment-hierarchy.svg)

- **ReplicaSet** : garantit qu'un nombre donné de réplicas d'un pod tourne à tout moment.
  Si un pod meurt, le ReplicaSet en recrée un. On ne le manipule quasiment jamais
  directement.
- **Deployment** : gère les ReplicaSets. À chaque changement de `template` (nouvelle image,
  nouvelle config), il crée un **nouveau** ReplicaSet et bascule progressivement les pods
  de l'ancien vers le nouveau.

### 🔀 Stratégies de déploiement

- **RollingUpdate** (par défaut) : remplace les pods progressivement, contrôlé par
  `maxUnavailable` et `maxSurge`. Zéro downtime si les probes readiness sont bien
  configurées.
- **Recreate** : tue tous les anciens pods avant de créer les nouveaux (downtime, mais
  utile si l'ancienne et la nouvelle version ne peuvent pas cohabiter, ex. migration DB
  incompatible).

> [!NOTE]
> **Blue/Green et Canary** ne sont **pas** des `strategy` natives du Deployment — Kubernetes
> ne connaît que RollingUpdate/Recreate. Elles se construisent par-dessus :
> - **Blue/Green** : deux Deployments complets (`blue` et `green`) tournent en parallèle, un
>   Service (ou l'Ingress) pointe sur l'un des deux via son `selector` ; le bascule est un
>   changement de label, instantané et facilement réversible, mais coûte 2x les ressources
>   pendant la transition.
> - **Canary** : un petit Deployment "canary" (ex. 1 réplica sur 20) partage le même `selector`
>   de Service que le Deployment principal — il reçoit une fraction du trafic **approximativement**
>   proportionnelle à son nombre de réplicas (sélection aléatoire du backend par kube-proxy en
>   mode iptables, voir [services-reseau.md](services-reseau.md) ; pas de pourcentage exact
>   configurable nativement). Un contrôle plus fin (pourcentage exact, routage par header) passe
>   par un Ingress controller avancé ou un service mesh (Istio, Linkerd).

### ⏮️ Rollback

Chaque révision d'un Deployment correspond à un ancien ReplicaSet conservé (scalé à 0) —
c'est lui qui porte le template de la révision (historique via `kubectl rollout history`).
`kubectl rollout undo` revient à une révision précédente en **re-scalant** ce ReplicaSet
existant.

> [!CAUTION]
> `rollout undo` ne peut **pas** recréer un ReplicaSet déjà supprimé. Si
> `revisionHistoryLimit` a nettoyé les anciens ReplicaSets, les révisions correspondantes
> n'existent plus et le retour arrière vers elles est impossible — il faut alors
> redéployer l'ancienne version depuis sa source (Git, chart Helm...).

> [!WARNING]
> - `replicas` définit le nombre désiré, mais le vrai contrôle du "combien de pods à la
>   fois" pendant une mise à jour vient de `maxSurge`/`maxUnavailable`.
> - Un Deployment sans `readinessProbe` correcte peut router du trafic vers des pods pas
>   encore prêts pendant un rollout.
> - `revisionHistoryLimit` contrôle combien d'anciens ReplicaSets sont gardés (pour
>   rollback) — par défaut 10.

## Références

- [Deployments (doc officielle)](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [ReplicaSet (doc officielle)](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
