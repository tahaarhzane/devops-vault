# Kubernetes

Notes techniques sur Kubernetes : concepts, manifests, opérations, dépannage.

## Sommaire

### Fondations

- [architecture.md](architecture.md) — control plane, worker nodes, reconciliation loop
- [pods.md](pods.md) — unité de base, cycle de vie, init containers

### Workloads

- [workloads-deployment.md](workloads-deployment.md) — Deployment, ReplicaSet, rolling update, rollback
- [workloads-statefulset-daemonset.md](workloads-statefulset-daemonset.md) — apps stateful, agents par nœud
- [workloads-job-cronjob.md](workloads-job-cronjob.md) — tâches ponctuelles et planifiées

### Réseau & configuration

- [services-reseau.md](services-reseau.md) — Service, types, DNS interne, kube-proxy
- [configmap-secret.md](configmap-secret.md) — injection de configuration et de secrets

### Stockage

- [stockage.md](stockage.md) — Volume, PV, PVC, StorageClass

### Scheduling & scaling

- [scheduling-avance.md](scheduling-avance.md) — requests/limits, affinity, taints/tolerations, HPA/VPA

### Observabilité

- [observabilite-probes.md](observabilite-probes.md) — startup/liveness/readiness, logs

### Organisation

- [namespaces.md](namespaces.md) — isolation logique, ResourceQuota, LimitRange

### Opérations

- [operations-troubleshooting.md](operations-troubleshooting.md) — maintenance de nœuds, rollout, diagnostic d'un pod

## Ordre de lecture conseillé

Pour découvrir Kubernetes de zéro : **architecture → pods → workloads-deployment →
services-reseau → stockage → configmap-secret**, puis le reste selon le besoin
(scheduling-avance, observabilite-probes, namespaces, operations-troubleshooting en
référence).

Les concepts transverses à plusieurs domaines (RBAC, Ingress, NetworkPolicy) sont documentés
dans [Concepts transverses](../Concepts%20transverses/README.md).
