---
titre: Exemple 3 — apiVersion par kind
domaine: Kubernetes
tags: [kubernetes, exemple, apiversion, api, reference]
derniere_revision: 2026-09-26
---

# Exemple 3 — apiVersion par kind

## Contexte

Troisième exemple du [fil rouge](README.md). Chaque manifeste commence par le couple
`apiVersion` / `kind`, et se tromper de version est l'une des premières causes d'échec d'un
`kubectl apply` (`no matches for kind "X" in version "Y"`). Cette page sert de référence :
quelle `apiVersion` écrire pour chaque `kind` courant, avec l'objet correspondant de
l'application `catalogue-api`.

## Notes

### 🔍 Lire une apiVersion

Une `apiVersion` a la forme **`<groupe>/<version>`** :

| Exemple | Groupe | Version | Remarque |
|---|---|---|---|
| `v1` | *core* (groupe vide) | `v1` | le groupe historique n'a pas de préfixe : on écrit seulement la version |
| `apps/v1` | `apps` | `v1` | workloads |
| `networking.k8s.io/v1` | `networking.k8s.io` | `v1` | les groupes récents ont un nom de domaine |
| `monitoring.coreos.com/v1` | `monitoring.coreos.com` | `v1` | groupe apporté par une CRD (ici Prometheus Operator) |

Niveaux de version :

| Niveau | Forme | Stabilité |
|---|---|---|
| **GA (stable)** | `v1`, `v2` | peut être dépréciée, mais n'est jamais retirée au sein d'une même version majeure de Kubernetes (1.x) |
| **Beta** | `v1beta1`, `v2beta2` | peut changer ; les nouvelles API beta ne sont plus activées par défaut depuis Kubernetes 1.24 |
| **Alpha** | `v1alpha1` | désactivée par défaut, peut disparaître sans préavis |

> [!NOTE]
> L'`apiVersion` décrit le **schéma** avec lequel on lit ou écrit l'objet, pas l'objet
> lui-même. L'apiserver stocke un objet dans une seule version et le sert dans toutes les
> versions actives de son groupe : un Deployment créé autrefois en `apps/v1beta2` se lit
> aujourd'hui en `apps/v1` sans rien migrer dans etcd. Ce sont les **manifestes** qui doivent
> être mis à jour quand une ancienne version est retirée.

### 📚 Référence : kind → apiVersion

Versions à utiliser sur un cluster actuel (1.25 et plus récent). La dernière colonne
rattache chaque kind au [cas d'usage](README.md).

**Core : `v1`**

| Kind | apiVersion | Dans le fil rouge |
|---|---|---|
| Namespace | `v1` | `boutique` |
| Pod | `v1` | créés par le ReplicaSet et le StatefulSet, jamais écrits à la main ici |
| Service | `v1` | `catalogue-api` (port 80) et `postgres` (port 5432) |
| ConfigMap | `v1` | `catalogue-api-config` |
| Secret | `v1` | `catalogue-db-credentials` |
| ServiceAccount | `v1` | `catalogue-api` |
| PersistentVolumeClaim | `v1` | `data-postgres-0`, généré par `volumeClaimTemplates` |
| PersistentVolume | `v1` | provisionné automatiquement par la StorageClass |
| ResourceQuota | `v1` | plafond CPU/mémoire du namespace `boutique` |
| LimitRange | `v1` | requests/limits par défaut des conteneurs du namespace |

**Workloads**

| Kind | apiVersion | Dans le fil rouge |
|---|---|---|
| Deployment | `apps/v1` | `catalogue-api` ([exemple 1](01-deployment-annote.md)) |
| ReplicaSet | `apps/v1` | `catalogue-api-<hash>`, créé par le Deployment |
| StatefulSet | `apps/v1` | `postgres` |
| DaemonSet | `apps/v1` | hors fil rouge : agent de logs présent sur chaque nœud |
| Job | `batch/v1` | migration du schéma de la base avant une mise à jour de l'API |
| CronJob | `batch/v1` | `pg_dump` nocturne de la base |

**Réseau**

| Kind | apiVersion | Dans le fil rouge |
|---|---|---|
| Ingress | `networking.k8s.io/v1` | `catalogue-api`, hôte `catalogue.example.com` |
| IngressClass | `networking.k8s.io/v1` | classe du contrôleur Ingress (nginx, Traefik...) |
| NetworkPolicy | `networking.k8s.io/v1` | seuls les Pods de l'API peuvent joindre `postgres:5432` |
| EndpointSlice | `discovery.k8s.io/v1` | générées automatiquement pour chaque Service |

**Scaling et disponibilité**

| Kind | apiVersion | Dans le fil rouge |
|---|---|---|
| HorizontalPodAutoscaler | `autoscaling/v2` | `catalogue-api`, de 3 à 10 réplicas selon le CPU |
| PodDisruptionBudget | `policy/v1` | `catalogue-api`, `minAvailable: 2` pendant les drains de nœuds |
| PriorityClass | `scheduling.k8s.io/v1` | priorité plus haute pour `postgres` que pour l'API |

**Stockage**

| Kind | apiVersion | Dans le fil rouge |
|---|---|---|
| StorageClass | `storage.k8s.io/v1` | classe de disque du PVC `data-postgres-0` |
| VolumeAttachment | `storage.k8s.io/v1` | créé par le système quand le disque est attaché au nœud |

**Sécurité (RBAC)**

| Kind | apiVersion | Dans le fil rouge |
|---|---|---|
| Role / RoleBinding | `rbac.authorization.k8s.io/v1` | droits de déploiement de la CI limités au namespace `boutique` |
| ClusterRole / ClusterRoleBinding | `rbac.authorization.k8s.io/v1` | lecture seule sur tout le cluster pour l'équipe d'astreinte |

**Extension de l'API**

| Kind | apiVersion | Rôle |
|---|---|---|
| CustomResourceDefinition | `apiextensions.k8s.io/v1` | déclare un nouveau kind ([crd-operators.md](../crd-operators.md)) |
| ValidatingWebhookConfiguration / MutatingWebhookConfiguration | `admissionregistration.k8s.io/v1` | webhooks d'admission |
| ValidatingAdmissionPolicy | `admissionregistration.k8s.io/v1` | règles de validation en CEL, sans webhook (GA en 1.30) |
| Lease | `coordination.k8s.io/v1` | élection de leader des controllers |
| RuntimeClass | `node.k8s.io/v1` | choisir un runtime de conteneurs alternatif (gVisor, Kata) |

**Kinds apportés par des CRD** : ils n'existent que si le composant est installé.

| Kind | apiVersion | Installé par |
|---|---|---|
| ServiceMonitor | `monitoring.coreos.com/v1` | Prometheus Operator — scraper les métriques de `catalogue-api` |
| Certificate | `cert-manager.io/v1` | cert-manager — certificat TLS de `catalogue.example.com` |
| Gateway / HTTPRoute | `gateway.networking.k8s.io/v1` | CRD de la Gateway API, successeur d'Ingress |
| VerticalPodAutoscaler | `autoscaling.k8s.io/v1` | VPA, projet autoscaler de Kubernetes |

> [!IMPORTANT]
> `autoscaling/v1` existe toujours pour le HorizontalPodAutoscaler, mais ne sait scaler que
> sur le CPU. `autoscaling/v2` ajoute la mémoire, les métriques custom et le réglage du
> comportement (`behavior`). Écrire `v2` dans tout nouveau manifeste.

### 🗑️ Versions retirées : les pièges des vieux manifestes

Un manifeste copié d'un vieux tutoriel ou d'un vieux chart échoue dès que sa version a été
**retirée** (plus servie par l'apiserver).

| Kind | apiVersion retirée | Retirée en | Remplacer par |
|---|---|---|---|
| Deployment, DaemonSet, ReplicaSet | `extensions/v1beta1` | 1.16 | `apps/v1` |
| Deployment, StatefulSet | `apps/v1beta1` | 1.16 | `apps/v1` |
| Deployment, StatefulSet, DaemonSet, ReplicaSet | `apps/v1beta2` | 1.16 | `apps/v1` |
| NetworkPolicy | `extensions/v1beta1` | 1.16 | `networking.k8s.io/v1` |
| Ingress | `extensions/v1beta1`, `networking.k8s.io/v1beta1` | 1.22 | `networking.k8s.io/v1` |
| CustomResourceDefinition | `apiextensions.k8s.io/v1beta1` | 1.22 | `apiextensions.k8s.io/v1` |
| Role, ClusterRole et bindings | `rbac.authorization.k8s.io/v1beta1` | 1.22 | `rbac.authorization.k8s.io/v1` |
| Validating/MutatingWebhookConfiguration | `admissionregistration.k8s.io/v1beta1` | 1.22 | `admissionregistration.k8s.io/v1` |
| CronJob | `batch/v1beta1` | 1.25 | `batch/v1` |
| PodDisruptionBudget | `policy/v1beta1` | 1.25 | `policy/v1` |
| PodSecurityPolicy | `policy/v1beta1` | 1.25 | aucun équivalent : Pod Security Admission ([securite-pod.md](../securite-pod.md)) |
| EndpointSlice | `discovery.k8s.io/v1beta1` | 1.25 | `discovery.k8s.io/v1` |
| HorizontalPodAutoscaler | `autoscaling/v2beta1` | 1.25 | `autoscaling/v2` |
| HorizontalPodAutoscaler | `autoscaling/v2beta2` | 1.26 | `autoscaling/v2` |

> [!WARNING]
> - **Changer la version ne suffit pas toujours** : le schéma peut avoir changé avec elle.
>   L'Ingress `networking.k8s.io/v1` exige `pathType` et remplace `serviceName`/`servicePort`
>   par `service.name`/`service.port.number`. Une CRD `v1` exige un schéma OpenAPI
>   structurel.
> - **Même schéma, autre comportement** : en `policy/v1`, un PodDisruptionBudget avec un
>   `selector` vide sélectionne **tous** les Pods du namespace ; en `policy/v1beta1`, il n'en
>   sélectionnait aucun.
> - **`Endpoints` (v1) est déprécié depuis 1.33** au profit d'EndpointSlice. Il est toujours
>   servi, mais les outils qui le lisent directement doivent migrer.

### 🧭 Vérifier sur son propre cluster

La référence qui fait foi est **le cluster lui-même** : les versions servies dépendent de sa
version et des CRD installées.

```bash
# Tous les kinds connus, avec leur groupe/version et leur portée (namespaced ou non)
kubectl api-resources

# Filtrer un groupe
kubectl api-resources --api-group=apps

# Toutes les versions servies (une par ligne, ex. autoscaling/v1, autoscaling/v2)
kubectl api-versions

# Schéma d'un kind dans une version précise
kubectl explain horizontalpodautoscaler --api-version=autoscaling/v2
```

Extrait de `kubectl api-resources` :

```
NAME                       SHORTNAMES   APIVERSION             NAMESPACED   KIND
configmaps                 cm           v1                     true         ConfigMap
services                   svc          v1                     true         Service
deployments                deploy       apps/v1                true         Deployment
statefulsets               sts          apps/v1                true         StatefulSet
cronjobs                   cj           batch/v1               true         CronJob
horizontalpodautoscalers   hpa          autoscaling/v2         true         HorizontalPodAutoscaler
ingresses                  ing          networking.k8s.io/v1   true         Ingress
storageclasses             sc           storage.k8s.io/v1      false        StorageClass
```

> [!TIP]
> - Avant une montée de version du cluster, **pluto** ou **kubent** (kube-no-trouble)
>   scannent manifestes, charts et Releases Helm à la recherche d'`apiVersion` dépréciées ou
>   retirées dans la version cible.
> - Dans un chart Helm, `.Capabilities.APIVersions.Has "policy/v1/PodDisruptionBudget"`
>   permet de choisir la version selon le cluster cible.
> - **kubeconform** valide des manifestes contre le schéma d'une version précise de
>   Kubernetes (`-kubernetes-version 1.33.0`), hors cluster, idéal en CI. Les manifestes des
>   exemples 1 et 2 passent cette validation.

## Références

- [Kubernetes API Overview](https://kubernetes.io/docs/reference/using-api/)
- [Deprecated API Migration Guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
- [Kubernetes Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
- [API reference (tous les kinds)](https://kubernetes.io/docs/reference/kubernetes-api/)
