---
titre: Helm
domaine: Kubernetes
tags: [kubernetes, helm, chart, packaging, templating]
derniere_revision: 2026-09-24
---

# Helm

## Contexte

Gérer des applications complexes avec des manifestes YAML bruts devient vite ingérable :
répétition entre environnements, pas de notion de version d'un déploiement complet, pas de
variables. Helm est le gestionnaire de paquets de facto pour Kubernetes — l'équivalent
d'`apt`/`yum` mais pour des applications K8s.

## Notes

### 📦 Chart : le paquet

Un **Chart** est un paquet Helm : un dossier avec une structure standard.

```
mon-app/
├── Chart.yaml          # métadonnées : nom, version du chart, appVersion
├── values.yaml          # valeurs par défaut, injectées dans les templates
├── templates/            # manifestes K8s sous forme de templates Go
│   ├── deployment.yaml
│   ├── service.yaml
│   └── _helpers.tpl     # fonctions/partiels réutilisables
└── charts/                # sous-charts (dépendances)
```

Les templates utilisent la syntaxe **Go templates** (+ fonctions Sprig) :

```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 🔁 Chart → Release → Cluster

![Chart, Release et cluster Kubernetes](assets/helm-chart-release.svg)

- `helm install <name> <chart>` : fusionne `values.yaml` (+ overrides) dans les templates,
  produit des manifestes K8s, les envoie à l'apiserver. Cette instance déployée s'appelle une
  **Release**.
- `helm upgrade <name> <chart>` : nouvelle révision de la Release, garde l'historique
  (`helm history <name>`).
- `helm rollback <name> <revision>` : revient à une révision précédente.
- `helm uninstall <name>` : supprime toutes les ressources créées par la Release.

> [!NOTE]
> Une même Release est **scoped à un namespace** — le même Chart peut être installé
> plusieurs fois sous des noms de Release différents (multi-tenant, multi-environnement).

### 🔧 Personnaliser les valeurs

```bash
helm install mon-app ./mon-app -f values-prod.yaml --set replicaCount=5
```

Ordre de priorité (le dernier gagne) : `values.yaml` du chart → fichiers `-f` (dans l'ordre
donné) → `--set` en ligne de commande.

### 📚 Dépôts (repos)

Un Chart peut être publié dans un **repo Helm** (registre HTTP ou OCI) :

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-redis bitnami/redis
```

### ✅ Bonnes pratiques

> [!TIP]
> - `helm lint ./mon-app` : valide la structure et la syntaxe du chart avant déploiement.
> - `helm template ./mon-app` : rend les manifestes localement sans rien envoyer au cluster —
>   indispensable pour relire ce qui sera réellement appliqué.
> - `helm diff upgrade` (plugin) : preview des changements avant un `upgrade` réel.
> - `--atomic` sur `install`/`upgrade` : rollback automatique si le déploiement échoue.
> - Séparer les valeurs par environnement (`values-dev.yaml`, `values-prod.yaml`) plutôt que
>   dupliquer des charts entiers.

Pour comparer Helm aux autres façons de déployer sur Kubernetes (manifeste brut, Kustomize,
Terraform), voir [deploiement-iac.md](deploiement-iac.md).

## Références

- [Helm (documentation officielle)](https://helm.sh/docs/)
- [Chart Template Guide](https://helm.sh/docs/chart_template_guide/getting_started/)
