---
titre: ConfigMap & Secret
domaine: Kubernetes
tags: [kubernetes, configmap, secret, configuration]
derniere_revision: 2026-09-23
---

# ConfigMap & Secret

## Contexte

Découpler la configuration du code de l'application (12-factor app) : ConfigMap pour les
données non sensibles, Secret pour les données sensibles (mots de passe, tokens, certs).
Structurellement identiques (clé/valeur), traités différemment en termes de sécurité.

## Notes

### 🗂️ ConfigMap

Stocke des paires clé/valeur consommables par les pods de trois façons :
- **variables d'environnement** (`envFrom` ou `env.valueFrom.configMapKeyRef`)
- **fichier monté en volume** (chaque clé devient un fichier, utile pour des fichiers de
  config type `application.yaml`, `nginx.conf`)
- **arguments de ligne de commande** (via variables d'env interpolées)

### 🔐 Secret

Même mécanique que ConfigMap, mais :
- valeurs encodées en **base64** dans l'objet
- stocké dans etcd — nécessite **chiffrement au repos d'etcd** activé côté cluster pour une
  vraie protection
- accès contrôlable plus finement via RBAC (`get`/`list` sur `secrets` séparé du reste)
- types spécialisés : `kubernetes.io/tls` (cert+clé), `kubernetes.io/dockerconfigjson`
  (credentials registry), `Opaque` (générique)

> [!CAUTION]
> Le base64 n'est **pas du chiffrement**, juste un encodage réversible en une commande
> (`base64 -d`). Ne jamais confondre avec de la sécurité réelle : quiconque a accès en
> lecture à l'objet (ou à etcd) lit le secret en clair.

### 🔁 Montage en volume vs variable d'environnement

| | Variable d'env | Volume monté |
|---|---|---|
| Mise à jour à chaud | non (nécessite redémarrage du pod) | oui (kubelet resynchronise le fichier, l'appli doit le relire) — **sauf** montage avec `subPath` |
| Surface d'exposition de la valeur | large (voir ci-dessous) | limitée au fichier monté |
| Adapté aux gros fichiers de config | non | oui |

`kubectl describe pod` ne montre **que la référence** au Secret (`secretKeyRef`), jamais sa
valeur. Les vrais risques d'exposition d'une variable d'environnement sont ailleurs :
- `/proc/<pid>/environ` : l'environnement du process est lisible par quiconque a accès au
  conteneur (ou au nœud).
- **Héritage** : chaque process enfant lancé par l'appli reçoit une copie de tout
  l'environnement, secrets compris.
- **Dumps** : beaucoup d'applis/frameworks écrivent leurs variables d'environnement dans les
  logs ou les rapports de crash.

→ Pour des secrets sensibles, préférer le montage en volume (moins de surface d'exposition
accidentelle) et une appli qui watch le fichier pour recharger sans redémarrer.

> [!WARNING]
> Un ConfigMap/Secret monté avec `subPath` (un seul fichier monté à un chemin précis) ne
> reçoit **jamais** les mises à jour à chaud — le fichier reste figé à la valeur du démarrage
> du pod. Il faut redémarrer le pod pour prendre en compte le changement.

### ✅ Bonnes pratiques et limites

> [!TIP]
> - Ne jamais committer de Secret en clair dans Git — utiliser un outil de gestion externe
>   (Azure Key Vault + CSI driver, Sealed Secrets, External Secrets Operator...).
> - Un ConfigMap/Secret monté en volume est mis à jour automatiquement par kubelet
>   (asynchrone, délai de quelques dizaines de secondes), mais **aucun redémarrage
>   automatique** du pod ni rechargement de l'appli n'est déclenché — à gérer côté
>   application ou via un mécanisme externe (ex. Reloader).
> - Taille max d'un ConfigMap/Secret : 1 MiB (limite etcd).
> - Immutabilité (`immutable: true`) recommandée pour les configs qui ne changent jamais après
>   déploiement : évite le watch permanent par kubelet et réduit la charge sur l'apiserver.

## Références

- [ConfigMaps (doc officielle)](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets (doc officielle)](https://kubernetes.io/docs/concepts/configuration/secret/)
