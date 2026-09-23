---
titre: Déployer sur Kubernetes : manifestes, Kustomize, Helm, Terraform
domaine: Kubernetes
tags: [kubernetes, iac, terraform, helm, kustomize, manifest]
derniere_revision: 2026-09-24
---

# Déployer sur Kubernetes : manifestes, Kustomize, Helm, Terraform

## Contexte

Il existe plusieurs formats/outils pour décrire et envoyer des ressources vers un cluster
Kubernetes. Tous finissent par parler au **même apiserver** — ce qui change, c'est le
format du code, la gestion des variations (dev/prod), et si un état est suivi en dehors du
cluster.

## Notes

### 🗺️ Vue d'ensemble

![Quatre chemins vers l'apiserver](assets/deploiement-vers-k8s.svg)

| Outil | Format | Templating / variantes | État suivi ? |
|---|---|---|---|
| Manifeste brut | YAML | aucun (copier-coller ou `envsubst` maison) | non (juste l'état du cluster) |
| Kustomize | YAML (base + patches) | overlays déclaratifs, pas de logique conditionnelle | non |
| Helm | YAML + templates Go | variables (`values.yaml`), conditions, boucles | oui (Release, `helm history`) |
| Terraform | HCL (`.tf`) | variables, modules, conditions | oui (state file) |

### 📄 Manifeste YAML brut

Le socle de tout : `apiVersion`, `kind`, `metadata`, `spec`.

```bash
kubectl apply -f deployment.yaml
```

`kubectl apply` (contrairement à `create`) fait un **merge à trois voies** (three-way merge)
entre la dernière config appliquée (annotation `kubectl.kubernetes.io/last-applied-configuration`),
l'état désiré du fichier, et l'état réel du cluster — permet des mises à jour incrémentales
sans écraser des champs gérés ailleurs (ex. par un autoscaler).

### 🧩 Kustomize

Composition de manifestes **sans templating** : une base commune + des patches par
environnement (`overlays/dev`, `overlays/prod`).

```
base/
  deployment.yaml
  kustomization.yaml
overlays/
  prod/
    kustomization.yaml   # patches : plus de réplicas, autre tag d'image...
```

```bash
kubectl apply -k overlays/prod
```

Intégré nativement à `kubectl` depuis la 1.14 — pas d'outil externe à installer. Volontairement
plus limité que Helm (pas de conditions/boucles) : la philosophie est de patcher du YAML valide,
pas de générer du YAML depuis un langage de template.

### ⚓ Helm

Voir [helm.md](helm.md) pour le détail — Chart + `values.yaml` templatés, avec suivi de
version (Release) et rollback intégré.

```bash
helm install mon-app ./chart -f values-prod.yaml
```

### ☁️ Terraform

Décrit l'infrastructure en **HCL** (`.tf`), pas limité à Kubernetes — peut gérer dans le même
projet le cluster AKS lui-même **et** les ressources qui tournent dedans, via deux providers :

- **provider `kubernetes`** : ressources K8s natives (`kubernetes_deployment`,
  `kubernetes_config_map`...), un bloc HCL par objet.
- **provider `helm`** : pilote l'installation de Charts Helm depuis Terraform
  (`helm_release`), combine les deux mondes.

```hcl
resource "helm_release" "mon_app" {
  name       = "mon-app"
  chart      = "./chart"
  namespace  = "prod"
  values     = [file("values-prod.yaml")]
}
```

```bash
terraform plan   # preview des changements
terraform apply  # applique, met à jour le state
```

Terraform garde un **state file** qui reflète ce qu'il croit avoir créé — c'est sa source de
vérité, distincte de l'état réel du cluster.

### ⚠️ Points de vigilance

> [!WARNING]
> - Ne jamais gérer les **mêmes ressources** avec deux outils différents (ex. Terraform +
>   `kubectl apply` manuel sur le même Deployment) — chacun écrase les changements de l'autre
>   à son prochain passage.
> - **Drift Terraform** : si quelqu'un modifie une ressource directement (`kubectl edit`) sans
>   passer par Terraform, le state diverge du cluster réel — `terraform plan` le détecte mais
>   ne corrige rien tant que `apply` n'est pas relancé.
> - Kustomize n'a pas de notion de version/rollback intégrée comme Helm — c'est juste du YAML
>   patché, l'historique dépend entièrement de Git.
> - Combiner Helm + Kustomize (post-render) est possible mais ajoute une couche de complexité
>   à documenter clairement si utilisé.

## Références

- [kubectl apply](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Kustomize (documentation officielle)](https://kubectl.docs.kubernetes.io/references/kustomize/)
- [Terraform provider kubernetes](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)
- [Terraform provider helm](https://registry.terraform.io/providers/hashicorp/helm/latest/docs)
