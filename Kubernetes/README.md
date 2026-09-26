# Kubernetes

Notes techniques sur Kubernetes : concepts, manifests, opérations, dépannage.

## Sommaire

### 🏗️ Fondations

- [architecture.md](architecture.md) — control plane, worker nodes, reconciliation loop
- [pods.md](pods.md) — unité de base, cycle de vie, init containers

### 📦 Workloads

- [workloads-deployment.md](workloads-deployment.md) — Deployment, ReplicaSet, rolling update, rollback
- [workloads-statefulset-daemonset.md](workloads-statefulset-daemonset.md) — apps stateful, agents par nœud
- [workloads-job-cronjob.md](workloads-job-cronjob.md) — tâches ponctuelles et planifiées

### 🌐 Réseau & configuration

- [services-reseau.md](services-reseau.md) — Service, types, DNS interne, kube-proxy
- [configmap-secret.md](configmap-secret.md) — injection de configuration et de secrets

### 🗄️ Stockage

- [stockage.md](stockage.md) — Volume, PV, PVC, StorageClass

### 📈 Scheduling & scaling

- [scheduling-avance.md](scheduling-avance.md) — requests/limits, affinity, taints/tolerations, HPA/VPA

### 💓 Observabilité

- [observabilite-probes.md](observabilite-probes.md) — startup/liveness/readiness, logs

### 🏷️ Organisation

- [namespaces.md](namespaces.md) — isolation logique, ResourceQuota, LimitRange

### 🛠️ Opérations

- [operations-troubleshooting.md](operations-troubleshooting.md) — maintenance de nœuds, rollout, diagnostic d'un pod

### 🚀 Déploiement & Infrastructure as Code

- [helm.md](helm.md) — Chart, Release, templating, bonnes pratiques
- [deploiement-iac.md](deploiement-iac.md) — comparatif manifeste YAML / Kustomize / Helm / Terraform

### 🔐 Sécurité & extensibilité

- [securite-pod.md](securite-pod.md) — SecurityContext, Pod Security Admission, ServiceAccount
- [crd-operators.md](crd-operators.md) — étendre l'API Kubernetes avec des types custom

### 🧪 Exemples concrets

Fil rouge unique : une API web `catalogue-api` et sa base PostgreSQL — [présentation du cas](exemples/README.md).

- [exemples/01-deployment-annote.md](exemples/01-deployment-annote.md) — Deployment complet, chaque champ annoté
- [exemples/02-deployment-helm.md](exemples/02-deployment-helm.md) — le même Deployment en chart Helm, correspondance values → template → rendu
- [exemples/03-apiversion-par-kind.md](exemples/03-apiversion-par-kind.md) — référence des apiVersion par kind, versions retirées

## Ordre de lecture conseillé

Pour découvrir Kubernetes de zéro : **architecture → pods → workloads-deployment →
services-reseau → stockage → configmap-secret**, puis le reste selon le besoin
(scheduling-avance, observabilite-probes, namespaces, operations-troubleshooting en
référence). Les [exemples concrets](exemples/README.md) se lisent une fois
workloads-deployment et helm connus.

Les concepts transverses à plusieurs domaines (RBAC, Ingress, NetworkPolicy) sont documentés
dans [Concepts transverses](../Concepts%20transverses/README.md).
