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

### ConfigMap

Stocke des paires clé/valeur consommables par les pods de trois façons :
- **variables d'environnement** (`envFrom` ou `env.valueFrom.configMapKeyRef`)
- **fichier monté en volume** (chaque clé devient un fichier, utile pour des fichiers de
  config type `application.yaml`, `nginx.conf`)
- **arguments de ligne de commande** (via variables d'env interpolées)

### Secret

Même mécanique que ConfigMap, mais :
- valeurs encodées en **base64** dans l'objet (⚠️ **pas chiffré**, juste encodé — ne pas
  confondre avec de la sécurité réelle)
- stocké dans etcd — nécessite **chiffrement au repos d'etcd** activé côté cluster pour une
  vraie protection
- accès contrôlable plus finement via RBAC (`get`/`list` sur `secrets` séparé du reste)
- types spécialisés : `kubernetes.io/tls` (cert+clé), `kubernetes.io/dockerconfigjson`
  (credentials registry), `Opaque` (générique)

### Montage en volume vs variable d'environnement

| | Variable d'env | Volume monté |
|---|---|---|
| Mise à jour à chaud | non (nécessite redémarrage du pod) | oui (kubelet resynchronise le fichier, l'appli doit le relire) |
| Visible dans `kubectl describe pod` / logs de crash | oui (risque de fuite en cas de dump) | non |
| Adapté aux gros fichiers de config | non | oui |

→ Pour des secrets sensibles, préférer le montage en volume (moins de surface d'exposition
accidentelle) et une appli qui watch le fichier pour recharger sans redémarrer.

### Bonnes pratiques et limites

- Ne jamais committer de Secret en clair dans Git — utiliser un outil de gestion externe
  (Azure Key Vault + CSI driver, Sealed Secrets, External Secrets Operator...).
- Un ConfigMap/Secret monté en volume est mis à jour automatiquement par kubelet
  (asynchrone, délai de quelques dizaines de secondes), mais **aucun redémarrage
  automatique** du pod ni rechargement de l'appli n'est déclenché — à gérer côté
  application ou via un mécanisme externe (ex. Reloader).
- Taille max d'un ConfigMap/Secret : 1 MiB (limite etcd).
- Immutabilité (`immutable: true`) recommandée pour les configs qui ne changent jamais après
  déploiement : évite le watch permanent par kubelet et réduit la charge sur l'apiserver.

## Références

- [ConfigMaps (doc officielle)](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Secrets (doc officielle)](https://kubernetes.io/docs/concepts/configuration/secret/)
