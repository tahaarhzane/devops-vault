---
titre: Exemple 1 — Deployment annoté ligne par ligne
domaine: Kubernetes
tags: [kubernetes, exemple, deployment, manifeste, yaml]
derniere_revision: 2026-09-26
---

# Exemple 1 — Deployment annoté ligne par ligne

## Contexte

Premier exemple du [fil rouge](README.md) : le Deployment de l'API `catalogue-api`, écrit
à la main en YAML brut, tel qu'on le passerait à `kubectl apply`. Chaque champ est commenté
pour dire **ce qu'il fait** et, quand il existe, **sa valeur par défaut**.

Le manifeste est volontairement complet (probes, sécurité, répartition) : c'est la base
qu'on retrouvera à l'identique, transformée en chart, dans
[l'exemple 2](02-deployment-helm.md).

## Notes

### 📄 Le manifeste complet

Fichier `catalogue-api-deployment.yaml` :

```yaml
# ─── Identité de l'objet ─────────────────────────────────────────────────────
apiVersion: apps/v1                # groupe d'API "apps", version "v1" (stable) — voir l'exemple 3
kind: Deployment                   # type d'objet → pris en charge par le Deployment controller
metadata:
  name: catalogue-api              # unique pour ce kind dans le namespace ; préfixe des ReplicaSets et Pods créés
  namespace: boutique              # si omis : namespace du contexte kubectl courant (souvent "default")
  labels:                          # labels DU DEPLOYMENT (pas des Pods) : tri et recherche (kubectl get -l ...)
    app.kubernetes.io/name: catalogue-api      # nom de l'application (labels recommandés Kubernetes)
    app.kubernetes.io/component: api           # rôle dans l'architecture
    app.kubernetes.io/part-of: boutique        # application globale dont elle fait partie
    app.kubernetes.io/version: "2.4.1"         # une valeur de label est une chaîne : guillemets par sécurité
  annotations:                     # métadonnées libres, non sélectionnables
    kubernetes.io/change-cause: "Image 2.4.1 : pagination du catalogue"   # colonne CHANGE-CAUSE de kubectl rollout history

# ─── Comportement du Deployment ──────────────────────────────────────────────
spec:
  replicas: 3                      # nombre de Pods désiré (défaut 1) ; à omettre si un HPA pilote le scaling
  revisionHistoryLimit: 5          # anciens ReplicaSets conservés pour rollback (défaut 10)
  minReadySeconds: 10              # un Pod doit rester Ready 10 s avant de compter comme "available" (défaut 0)
  progressDeadlineSeconds: 600     # sans progrès pendant 600 s → condition Progressing=False (défaut 600) ; pas de rollback auto
  selector:                        # quels Pods appartiennent à ce Deployment — IMMUABLE après création
    matchLabels:                   # doit correspondre à template.metadata.labels, sinon l'apiserver refuse l'objet
      app.kubernetes.io/name: catalogue-api
      app.kubernetes.io/component: api
  strategy:
    type: RollingUpdate            # remplacement progressif (défaut) ; seule autre valeur : Recreate
    rollingUpdate:
      maxSurge: 1                  # Pods en plus de replicas autorisés pendant le rollout (défaut 25 %)
      maxUnavailable: 0            # Pods en moins tolérés (défaut 25 %) ; 0 = capacité jamais réduite

  # ─── Modèle de Pod : toute modification ici crée un nouveau ReplicaSet (= rollout) ───
  template:
    metadata:
      labels:                      # labels portés par chaque Pod : ciblés par selector ET par le Service
        app.kubernetes.io/name: catalogue-api
        app.kubernetes.io/component: api
        app.kubernetes.io/part-of: boutique
        app.kubernetes.io/version: "2.4.1"
    spec:
      serviceAccountName: catalogue-api       # identité du Pod auprès de l'apiserver (défaut : "default")
      automountServiceAccountToken: false     # l'API n'appelle jamais l'apiserver → aucun token monté
      terminationGracePeriodSeconds: 30       # délai entre SIGTERM et SIGKILL à l'arrêt du Pod (défaut 30)

      securityContext:                        # niveau Pod : s'applique à tous les conteneurs
        runAsNonRoot: true                    # le kubelet refuse de démarrer un conteneur dont l'UID serait 0
        runAsUser: 10001                      # UID imposé (sinon : l'USER défini dans l'image)
        runAsGroup: 10001                     # GID principal des processus
        seccompProfile:
          type: RuntimeDefault                # filtre d'appels système du runtime ; exigé par le niveau PSA "restricted"

      topologySpreadConstraints:              # répartir les réplicas pour survivre à la perte d'un nœud
        - maxSkew: 1                          # écart maximal de Pods entre deux nœuds
          topologyKey: kubernetes.io/hostname # domaine de répartition = le nœud (topology.kubernetes.io/zone pour les zones)
          whenUnsatisfiable: ScheduleAnyway   # simple préférence ; DoNotSchedule en ferait une contrainte dure
          labelSelector:                      # quels Pods compter pour mesurer l'écart
            matchLabels:
              app.kubernetes.io/name: catalogue-api
              app.kubernetes.io/component: api

      containers:
        - name: api                           # nom du conteneur dans le Pod (kubectl logs <pod> -c api)
          image: registry.example.com/catalogue-api:2.4.1   # tag fixe : jamais :latest en production
          imagePullPolicy: IfNotPresent       # défaut si le tag n'est pas :latest (Always si :latest ou sans tag)
          ports:
            - name: http                      # nom réutilisé par les probes et le Service (targetPort: http)
              containerPort: 8080             # documentaire : l'application écoute même si le port n'est pas déclaré
              protocol: TCP                   # défaut TCP

          envFrom:
            - configMapRef:
                name: catalogue-api-config    # chaque clé du ConfigMap devient une variable (DB_HOST, DB_PORT...)
          env:                                # variables unitaires ; prioritaires sur envFrom en cas de doublon
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: catalogue-db-credentials   # Secret créé hors de ce fichier (voir plus bas)
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalogue-db-credentials
                  key: password

          resources:
            requests:                         # réservé par le scheduler pour choisir un nœud
              cpu: 250m                       # 250 millicores = un quart de cœur
              memory: 256Mi
            limits:                           # plafond : CPU → throttling ; mémoire → OOMKilled
              cpu: 500m
              memory: 512Mi                   # requests ≠ limits : pas Guaranteed → QoS Burstable

          startupProbe:                       # protège le démarrage : liveness/readiness ne tournent qu'après son succès
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 5
            failureThreshold: 24              # 24 × 5 s = jusqu'à 120 s pour démarrer, sinon redémarrage du conteneur
          readinessProbe:                     # échec → Pod retiré des endpoints du Service (PAS de redémarrage)
            httpGet:
              path: /ready                    # vérifie aussi que PostgreSQL répond
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:                      # échec → le kubelet redémarre le conteneur
            httpGet:
              path: /healthz                  # ne doit PAS dépendre de PostgreSQL (voir l'encadré plus bas)
              port: http
            periodSeconds: 20
            failureThreshold: 3

          securityContext:                    # niveau conteneur : complète celui du Pod
            allowPrivilegeEscalation: false   # pose no_new_privs : un binaire setuid ne peut pas élever les droits
            readOnlyRootFilesystem: true      # système de fichiers de l'image en lecture seule
            capabilities:
              drop: ["ALL"]                   # aucune capability Linux (8080 > 1024 : pas besoin de NET_BIND_SERVICE)
          volumeMounts:
            - name: tmp
              mountPath: /tmp                 # seul répertoire inscriptible du conteneur

      volumes:
        - name: tmp
          emptyDir: {}                        # vide à la création du Pod, supprimé avec lui
```

> [!IMPORTANT]
> `spec.selector` est **immuable** : une fois le Deployment créé, le modifier est refusé
> (`field is immutable`). Il faut supprimer puis recréer le Deployment. C'est pour ça qu'on
> n'y met que des labels stables (`name`, `component`) et **jamais** la version.

### 📋 Défauts et choix faits ici

| Champ | Défaut Kubernetes | Valeur choisie | Pourquoi |
|---|---|---|---|
| `replicas` | 1 | 3 | survivre à la perte d'un Pod ou d'un nœud |
| `revisionHistoryLimit` | 10 | 5 | assez pour revenir en arrière, moins d'objets inutiles |
| `minReadySeconds` | 0 | 10 | un Pod qui crashe juste après être devenu Ready ne valide pas le rollout |
| `progressDeadlineSeconds` | 600 | 600 | explicite pour la lecture |
| `strategy.rollingUpdate` | 25 % / 25 % | `maxSurge: 1`, `maxUnavailable: 0` | le rollout ne fait jamais descendre sous 3 Pods disponibles : un nouveau Pod doit être disponible avant qu'un ancien soit supprimé |
| `imagePullPolicy` | dépend du tag | `IfNotPresent` | tag immuable, inutile de retélécharger |
| `serviceAccountName` | `default` | `catalogue-api` | identité dédiée, sans token monté |
| `terminationGracePeriodSeconds` | 30 | 30 | explicite pour la lecture |
| `securityContext` | aucun | non-root, seccomp, FS en lecture seule, `drop: ALL` | Pod conforme au niveau `restricted` ([securite-pod.md](../securite-pod.md)) |

### 🧩 Les objets que ce Deployment référence

Le Deployment seul ne démarre pas : ses Pods restent en `CreateContainerConfigError` tant
que le ConfigMap et le Secret n'existent pas, et le Pod n'est même pas créé sans le
ServiceAccount. Fichier `catalogue-api-deps.yaml` :

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalogue-api
  namespace: boutique
automountServiceAccountToken: false   # défaut appliqué aux Pods qui utilisent ce compte
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogue-api-config
  namespace: boutique
data:                                 # map de chaînes uniquement
  DB_HOST: postgres                   # nom du Service PostgreSQL, résolu par le DNS du cluster (même namespace)
  DB_PORT: "5432"                     # guillemets obligatoires : un nombre nu est refusé dans data
  DB_NAME: catalogue
  LOG_LEVEL: info
---
apiVersion: v1
kind: Service
metadata:
  name: catalogue-api
  namespace: boutique
spec:
  type: ClusterIP                     # défaut : IP virtuelle interne au cluster
  selector:                           # mêmes labels que les Pods du Deployment
    app.kubernetes.io/name: catalogue-api
    app.kubernetes.io/component: api
  ports:
    - name: http
      port: 80                        # port exposé par le Service
      targetPort: http                # port NOMMÉ du conteneur → 8080
```

Le Secret est créé **à part**, jamais committé en clair dans Git (en production : External
Secrets, Sealed Secrets ou le coffre du cloud, voir [configmap-secret.md](../configmap-secret.md)) :

```bash
kubectl create secret generic catalogue-db-credentials -n boutique \
  --from-literal=username=catalogue \
  --from-literal=password="$(openssl rand -base64 24)"
```

### 🚀 Appliquer et vérifier

```bash
kubectl apply -f catalogue-api-deps.yaml -f catalogue-api-deployment.yaml
kubectl rollout status deployment/catalogue-api -n boutique
kubectl get deploy,rs,pods -n boutique -l app.kubernetes.io/name=catalogue-api
```

Sortie typique (les suffixes sont générés) :

```
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalogue-api   3/3     3            3           52s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/catalogue-api-6d8f7c9b4d   3         3         3       52s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/catalogue-api-6d8f7c9b4d-2xkqp   1/1     Running   0          52s
pod/catalogue-api-6d8f7c9b4d-h7m5z   1/1     Running   0          52s
pod/catalogue-api-6d8f7c9b4d-tq9wl   1/1     Running   0          52s
```

Le suffixe du ReplicaSet (`6d8f7c9b4d`) est le **pod-template-hash**, un hash du
`template`. Changer l'image produit un autre hash, donc un autre ReplicaSet : c'est le
rollout décrit dans [workloads-deployment.md](../workloads-deployment.md).

> [!TIP]
> - `kubectl apply --dry-run=server -f ...` : fait valider le manifeste par l'apiserver
>   (schéma, admission, Pod Security) sans rien créer.
> - `kubectl explain deployment.spec.strategy.rollingUpdate` : documentation de n'importe
>   quel champ, directement depuis l'API du cluster.
> - `kubectl diff -f ...` : ce qui changerait réellement avant un `apply`.

> [!WARNING]
> - **Liveness et base de données** : si `/healthz` testait PostgreSQL, une panne de la base
>   ferait échouer la liveness de tous les Pods, qui redémarreraient en boucle sans rien
>   réparer. La dépendance à la base va dans la **readiness** (Pod retiré du trafic, pas
>   redémarré).
> - **`replicas` et HPA** : si un HPA gère ce Deployment, laisser `replicas: 3` dans le
>   fichier fait qu'à chaque `kubectl apply` le nombre de Pods est remis à 3, puis corrigé
>   par le HPA. Supprimer le champ du manifeste dans ce cas.
> - **Variables d'environnement figées** : les valeurs issues du ConfigMap et du Secret sont
>   lues au démarrage du conteneur. Les modifier ne change rien aux Pods existants ; il faut
>   un `kubectl rollout restart deployment/catalogue-api -n boutique`.
> - **`progressDeadlineSeconds` n'annule rien** : un rollout bloqué est seulement signalé
>   (`Progressing=False`, `ProgressDeadlineExceeded`). Le retour arrière reste une action
>   explicite (`kubectl rollout undo`).

## Références

- [Deployments (doc officielle)](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
