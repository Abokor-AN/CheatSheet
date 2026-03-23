# Kubernetes Cheatsheet - Objets de Création

Référence complète pour la création de tous les objets Kubernetes, avec toutes les options disponibles et commentées.

## Fichiers

| Fichier | Objet(s) | Description |
|---------|----------|-------------|
| `01-pod.yaml` | **Pod** | Unité de base: conteneurs, volumes, probes, scheduling, sécurité |
| `02-deployment.yaml` | **Deployment** | Déploiement avec rolling updates, rollbacks, scaling |
| `03-service.yaml` | **Service** | ClusterIP, NodePort, LoadBalancer, ExternalName, Headless |
| `04-configmap-secret.yaml` | **ConfigMap / Secret** | Configuration et données sensibles, tous les types de Secrets |
| `05-ingress.yaml` | **Ingress / IngressClass** | Routage HTTP/HTTPS, TLS, annotations nginx, cert-manager |
| `06-statefulset.yaml` | **StatefulSet** | Applications stateful (DBs), PVC templates, DNS stable |
| `07-daemonset.yaml` | **DaemonSet** | Agent par nœud: monitoring, logs, proxies réseau |
| `08-job-cronjob.yaml` | **Job / CronJob** | Tâches batch, planification cron, parallelisme |
| `09-persistentvolume-pvc.yaml` | **PV / PVC / StorageClass** | Stockage persistant, snapshots, provisionnement dynamique |
| `10-rbac.yaml` | **RBAC** | ServiceAccount, Role, ClusterRole, RoleBinding |
| `11-networkpolicy.yaml` | **NetworkPolicy** | Contrôle du trafic réseau entre pods |
| `12-hpa-vpa-pdb.yaml` | **HPA / VPA / PDB** | Autoscaling horizontal/vertical, disruption budget |
| `13-namespace-resourcequota-limitrange.yaml` | **Namespace / ResourceQuota / LimitRange / PriorityClass** | Isolation, quotas, limites par défaut |
| `14-advanced-objects.yaml` | **RuntimeClass / PSA / Webhooks / CRD** | Objets avancés: runtimes, sécurité, extensions API |
| `15-replicaset-endpointslice-gateway.yaml` | **ReplicaSet / EndpointSlice / Gateway API** | Base des Deployments, successeur Endpoints, successeur Ingress |
| `16-helm.md` | **Helm** | Packaging, Chart.yaml, values, templates Go, hooks, CLI complète |
| `17-kustomize.md` | **Kustomize** | Overlays, patches (Strategic Merge, JSON), composants, vs Helm |
| `18-troubleshooting.md` | **Troubleshooting** | Tous les états d'erreur, debug pods/services/RBAC/DNS/PVC |
| `19-keda.yaml` | **KEDA** | ScaledObject, ScaledJob, tous les triggers (SQS, Kafka, Redis, Prometheus…) |
| `20-secrets-management.yaml` | **Gestion des Secrets** | Sealed Secrets, External Secrets Operator, Vault, encryption at rest |
| `21-kyverno-opa.yaml` | **Kyverno / OPA Gatekeeper** | Validate, Mutate, Generate policies, ConstraintTemplates Rego |
| `22-cluster-ops.md` | **Opérations Cluster** | kubeadm, upgrade, etcd backup/restore, certificats TLS, Velero |

## Commandes Essentielles

```bash
# Appliquer une config
kubectl apply -f <file>.yaml
kubectl apply -f kubernetes/          # Tout un dossier

# Inspecter
kubectl get <resource>
kubectl describe <resource> <name>
kubectl get <resource> -o yaml        # YAML complet

# Logs & Debug
kubectl logs <pod> -c <container> -f
kubectl exec -it <pod> -- /bin/sh
kubectl port-forward <pod> 8080:80
kubectl events --field-selector involvedObject.name=<name>

# Rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# Scaling
kubectl scale deployment <name> --replicas=5

# Context & Namespace
kubectl config set-context --current --namespace=<ns>
kubectl config get-contexts
```

## Ressources Kubernetes

| apiVersion | Ressources |
|------------|-----------|
| `v1` | Pod, Service, ConfigMap, Secret, Namespace, PersistentVolume, PersistentVolumeClaim, ServiceAccount, LimitRange, ResourceQuota |
| `apps/v1` | Deployment, StatefulSet, DaemonSet, ReplicaSet |
| `batch/v1` | Job, CronJob |
| `networking.k8s.io/v1` | Ingress, IngressClass, NetworkPolicy |
| `rbac.authorization.k8s.io/v1` | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| `storage.k8s.io/v1` | StorageClass, VolumeAttachment |
| `autoscaling/v2` | HorizontalPodAutoscaler |
| `autoscaling.k8s.io/v1` | VerticalPodAutoscaler *(CRD - VPA Operator)* |
| `policy/v1` | PodDisruptionBudget |
| `scheduling.k8s.io/v1` | PriorityClass |
| `node.k8s.io/v1` | RuntimeClass |
| `apiextensions.k8s.io/v1` | CustomResourceDefinition |
| `admissionregistration.k8s.io/v1` | MutatingWebhookConfiguration, ValidatingWebhookConfiguration |
