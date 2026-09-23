---
titre: Namespaces
domaine: Kubernetes
tags: [kubernetes, namespace, multi-tenant, resourcequota]
derniere_revision: 2026-09-23
---

# Namespaces

## Contexte

Mécanisme de **cloisonnement logique** d'un cluster en plusieurs "sous-clusters" virtuels.
Utile pour séparer des équipes, des environnements (dev/staging/prod dans un même cluster),
ou des applications.

## Notes

### Ce que le namespace isole

- **Noms d'objets** : deux objets peuvent avoir le même nom dans deux namespaces différents
  (le nom complet est en réalité `<nom>.<namespace>`)
- **Quotas de ressources** (via `ResourceQuota`)
- **RBAC** : les `RoleBinding` (par opposition à `ClusterRoleBinding`) s'appliquent à un
  namespace précis
- **NetworkPolicy** : peuvent restreindre le trafic entre namespaces

### Ce que le namespace n'isole PAS

- Les **nœuds** : pas de cloisonnement physique, les pods de namespaces différents peuvent
  tourner sur le même nœud et partager ses ressources CPU/mémoire réelles (sauf
  quotas/limits bien configurés)
- Les ressources **cluster-scoped** : Node, PersistentVolume, StorageClass, ClusterRole,
  Namespace lui-même — ces objets n'appartiennent à aucun namespace

### ResourceQuota et LimitRange

- **ResourceQuota** : plafonne la consommation totale d'un namespace (CPU/mémoire totale,
  nombre max de pods, PVC, Services...). Empêche une équipe de monopoliser le cluster.
- **LimitRange** : définit des valeurs par défaut ou des bornes min/max de
  requests/limits **par pod/conteneur** dans le namespace — évite les pods sans limites
  définies.

### Namespaces par défaut

- `default` : namespace utilisé si aucun n'est spécifié
- `kube-system` : composants internes de Kubernetes (CoreDNS, kube-proxy...)
- `kube-public` : lisible par tous, données publiques du cluster
- `kube-node-lease` : objets Lease pour la heartbeat des nœuds

### Points de vigilance

- Supprimer un namespace supprime **en cascade** tous les objets qu'il contient — action
  irréversible sans backup (etcd snapshot ou sauvegarde applicative).
- Sans `NetworkPolicy`, tous les pods peuvent communiquer entre eux **quel que soit le
  namespace** par défaut (le cloisonnement réseau n'est pas automatique).
- Un cluster multi-tenant sérieux combine namespace + RBAC + ResourceQuota + NetworkPolicy
  — le namespace seul n'est qu'une étiquette organisationnelle.

## Références

- [Namespaces (doc officielle)](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
