---
titre: Job & CronJob
domaine: Kubernetes
tags: [kubernetes, job, cronjob, batch]
derniere_revision: 2026-09-23
---

# Job & CronJob

## Contexte

Pour les traitements **finis** (batch, tâches ponctuelles ou planifiées), par opposition
aux workloads de service continu (Deployment, StatefulSet, DaemonSet).

## Notes

### Job

Crée un ou plusieurs pods et s'assure qu'un nombre défini d'entre eux se termine avec
succès (exit code 0). Contrairement à un Deployment, un pod terminé n'est **pas** relancé
en boucle — le Job est considéré complet.

Paramètres clés :
- `completions` : nombre total d'exécutions réussies attendues
- `parallelism` : nombre de pods exécutés en parallèle
- `backoffLimit` : nombre de tentatives avant de marquer le Job en échec
- `activeDeadlineSeconds` : timeout global du Job

Modes d'usage :
- **une tâche unique** (`completions` non défini) : un seul pod doit réussir
- **tâches parallèles à complétion fixe** : ex. traiter 10 fichiers, `completions: 10`
- **file de travail** (work queue) : les pods coordonnent eux-mêmes qui prend quelle tâche

### CronJob

Crée des Jobs selon un planning au format cron (`schedule: "*/5 * * * *"`). Utile pour :
- sauvegardes planifiées
- nettoyage périodique (purge de données, rotation de logs)
- rapports générés à intervalle régulier

Paramètres clés :
- `concurrencyPolicy` : `Allow` (défaut), `Forbid` (skip si le job précédent tourne encore),
  `Replace` (tue le job en cours et lance le nouveau)
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` : combien d'anciens Jobs garder
- `startingDeadlineSeconds` : marge de tolérance si le scheduler a raté l'heure planifiée
  (ex. cluster indisponible)

### Points de vigilance

- Un CronJob qui rate son créneau (cluster down) ne rattrape pas les exécutions manquées
  au-delà de `startingDeadlineSeconds`.
- Toujours borner `backoffLimit` et `activeDeadlineSeconds` pour éviter un Job qui boucle
  indéfiniment et consomme des ressources.
- Les pods de Jobs terminés restent visibles (`kubectl get pods`) jusqu'à nettoyage —
  penser au TTL controller (`ttlSecondsAfterFinished`) pour l'auto-nettoyage.

## Références

- [Jobs (doc officielle)](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [CronJob (doc officielle)](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
