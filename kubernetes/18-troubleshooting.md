# ============================================================
# KUBERNETES TROUBLESHOOTING - CHEATSHEET COMPLET
# ============================================================

## États des Pods et causes

| État | Cause probable | Solution |
|------|---------------|----------|
| `Pending` | Pas assez de ressources, scheduling impossible | Voir `kubectl describe pod` → Events |
| `ImagePullBackOff` | Image introuvable, credentials manquants | Vérifier nom image, imagePullSecrets |
| `ErrImagePull` | Erreur réseau ou registry inaccessible | Vérifier connectivité registry |
| `CrashLoopBackOff` | App plante au démarrage | Voir les logs |
| `OOMKilled` | Mémoire insuffisante (limits atteintes) | Augmenter memory limit |
| `Error` | Erreur d'exécution | Voir les logs |
| `Terminating` (bloqué) | Finalizers non résolus, volume bloqué | Supprimer finalizers manuellement |
| `ContainerCreating` | Volume non disponible, config manquante | Voir Events du pod |
| `Init:Error` | Init container a échoué | Voir logs de l'init container |
| `Init:CrashLoopBackOff` | Init container boucle | Voir logs de l'init container |
| `CreateContainerConfigError` | Secret/ConfigMap manquant | Vérifier les volumes et envFrom |
| `RunContainerError` | Problème de permission, commande invalide | Voir `kubectl describe pod` |
| `InvalidImageName` | Nom d'image mal formé | Corriger le nom dans le YAML |
| `Evicted` | Node en pression (memory/disk) | Vérifier ressources du nœud |
| `NodeLost` | Nœud inaccessible | Vérifier l'état du nœud |

---

## Commandes de diagnostic essentielles

```bash
# ─── POD ────────────────────────────────────────────────────

# Vue globale
kubectl get pods -A                           # Tous les namespaces
kubectl get pods -A --field-selector=status.phase!=Running
kubectl get pods -o wide                      # IP, nœud
kubectl get events --sort-by=.lastTimestamp   # Événements récents
kubectl get events -n my-ns --sort-by='.metadata.creationTimestamp'

# Détails complets (TOUJOURS commencer par là)
kubectl describe pod my-pod
kubectl describe pod my-pod -n my-ns

# Logs
kubectl logs my-pod                           # Logs du conteneur principal
kubectl logs my-pod -c my-container           # Conteneur spécifique
kubectl logs my-pod --previous                # Logs avant redémarrage
kubectl logs my-pod -f                        # Suivre en temps réel
kubectl logs my-pod --tail=100                # 100 dernières lignes
kubectl logs my-pod --since=1h               # Depuis 1 heure
kubectl logs -l app=my-app --all-containers  # Tous les pods du label
kubectl logs -l app=my-app --prefix          # Avec préfixe pod/container

# Shell interactif
kubectl exec -it my-pod -- /bin/sh
kubectl exec -it my-pod -c my-container -- /bin/bash
kubectl exec my-pod -- env                    # Lister les variables d'env
kubectl exec my-pod -- cat /etc/config/app.conf

# Debug avec pod éphémère (K8s 1.23+)
kubectl debug -it my-pod --image=nicolaka/netshoot --target=app
kubectl debug -it my-pod --image=busybox --copy-to=debug-pod
kubectl debug node/worker-1 -it --image=ubuntu  # Debug un nœud

# ─── NODE ───────────────────────────────────────────────────
kubectl get nodes
kubectl get nodes -o wide
kubectl describe node worker-1
kubectl top nodes                             # CPU/Memory utilisé
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory -A

# Drainage pour maintenance
kubectl cordon worker-1                       # Empêcher scheduling
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker-1                     # Réactiver scheduling

# ─── DEPLOYMENT ─────────────────────────────────────────────
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl get replicasets -l app=my-app -o wide

# ─── SERVICE / RÉSEAU ───────────────────────────────────────
kubectl get svc
kubectl describe svc my-service
kubectl get endpoints my-service             # Voir les pods cibles

# Test de connectivité depuis l'intérieur du cluster
kubectl run nettest --image=nicolaka/netshoot --rm -it -- bash
# Puis dans le shell:
#   curl http://my-service.my-ns.svc.cluster.local
#   nslookup my-service.my-ns.svc.cluster.local
#   nc -zv my-service 8080
#   dig my-service.my-ns.svc.cluster.local

# ─── RESOURCES / QUOTA ──────────────────────────────────────
kubectl describe resourcequota -n production
kubectl describe limitrange -n production
kubectl get pvc -A
kubectl describe pvc my-pvc

# ─── RBAC ───────────────────────────────────────────────────
kubectl auth can-i list pods -n production
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa
kubectl auth can-i '*' '*'           # Vérifier si cluster-admin

# ─── API SERVER ─────────────────────────────────────────────
kubectl api-resources                # Toutes les ressources disponibles
kubectl api-versions                 # Toutes les versions d'API
kubectl explain pod.spec.containers  # Documentation d'un champ
kubectl explain deployment --recursive
```

---

## Scénarios de debug courants

### Pod en `Pending`

```bash
kubectl describe pod my-pod
# Chercher dans Events:

# 1. "0/3 nodes are available: Insufficient cpu"
#    → Augmenter resources disponibles ou diminuer requests

# 2. "0/3 nodes are available: node(s) had untolerated taint"
#    → Ajouter tolerations ou retirer le taint

# 3. "0/3 nodes are available: node(s) didn't match Pod's node affinity/selector"
#    → Vérifier nodeSelector/affinity et labels des nœuds

# 4. "persistentvolumeclaim 'my-pvc' not found"
#    → Créer le PVC ou corriger le nom

# 5. "no persistent volumes available for this claim"
#    → Créer un PV ou attendre le provisionnement dynamique

kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory
kubectl describe nodes | grep -A5 "Allocated resources"
```

### Pod en `CrashLoopBackOff`

```bash
kubectl logs my-pod --previous        # Logs AVANT le crash
kubectl describe pod my-pod           # Voir Exit Code dans Last State

# Exit codes courants:
# 0   → Succès normal (ne devrait pas crash)
# 1   → Erreur générale de l'application
# 2   → Mauvaise utilisation des commandes shell
# 125 → Erreur Docker
# 126 → Permission denied (commande non exécutable)
# 127 → Commande non trouvée (CMD incorrect)
# 128 → Signal invalide
# 130 → SIGINT (Ctrl+C)
# 137 → SIGKILL (OOMKilled ou kill -9)
# 139 → SIGSEGV (segfault)
# 143 → SIGTERM
# 255 → Exit code hors limites

# Causes fréquentes:
# - Application plante au démarrage (config incorrecte, DB inaccessible)
# - readinessProbe trop stricte → pod tué avant qu'il soit prêt
# - OOMKilled (137) → augmenter memory limit
# - Command/Args incorrects dans le YAML

# Astuce: override command pour investiguer
kubectl run debug-pod --image=my-app:1.0 \
  --command -- sleep infinity
kubectl exec -it debug-pod -- /bin/sh
```

### Pod en `OOMKilled`

```bash
kubectl describe pod my-pod
# Chercher: "OOMKilled" et "Last State: Terminated Reason: OOMKilled"

# Solutions:
# 1. Augmenter la limit mémoire
kubectl set resources deployment my-app --limits=memory=512Mi

# 2. Activer VPA pour recommandations
kubectl describe vpa my-vpa           # Voir Target recommendation

# 3. Profiler l'application pour fuites mémoire
kubectl top pods --containers
kubectl exec my-pod -- cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

### Pod en `ImagePullBackOff`

```bash
kubectl describe pod my-pod
# Chercher dans Events: "Failed to pull image"

# 1. Image inexistante
docker pull my-registry/my-app:1.0   # Tester localement

# 2. Credentials manquants
kubectl get secret registry-creds -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# Créer le secret de registry
kubectl create secret docker-registry registry-creds \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword

# Ajouter au ServiceAccount
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "registry-creds"}]}'

# 3. Réseau/DNS
kubectl run dns-test --image=busybox --rm -it -- nslookup registry.example.com
```

### Pod en `CreateContainerConfigError`

```bash
kubectl describe pod my-pod
# Causes: ConfigMap ou Secret référencé manquant

# Vérifier
kubectl get configmap my-config -n my-ns
kubectl get secret my-secret -n my-ns

# Voir les références dans le pod
kubectl get pod my-pod -o jsonpath='{.spec.volumes}'
kubectl get pod my-pod -o jsonpath='{.spec.containers[0].envFrom}'
```

### Service ne route pas vers les pods

```bash
# 1. Vérifier que les endpoints existent
kubectl get endpoints my-service
# Si ENDPOINTS = <none>: le selector ne matche aucun pod

# 2. Comparer les labels
kubectl get svc my-service -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels -l app=my-app

# 3. Vérifier que les pods sont Ready
kubectl get pods -l app=my-app
# STATUS doit être Running, READY doit être 1/1 (ou N/N)

# 4. Vérifier les ports (targetPort doit matcher containerPort)
kubectl get svc my-service -o yaml | grep -A5 ports
kubectl get pod my-pod -o yaml | grep -A5 containerPort

# 5. Test depuis un autre pod
kubectl run curl-test --image=curlimages/curl --rm -it -- \
  curl -v http://my-service.my-ns.svc.cluster.local:80
```

### DNS ne résout pas

```bash
# 1. Tester la résolution DNS
kubectl run dns-test --image=busybox --rm -it -- nslookup kubernetes
kubectl run dns-test --image=busybox --rm -it -- nslookup my-service.my-ns.svc.cluster.local

# 2. Vérifier CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# 3. Vérifier la config CoreDNS
kubectl get configmap coredns -n kube-system -o yaml

# 4. Format DNS dans le cluster
# <service>.<namespace>.svc.cluster.local
# <pod-ip>.<namespace>.pod.cluster.local
# <pod-name>.<service>.<namespace>.svc.cluster.local  (StatefulSet)
```

### PVC en `Pending`

```bash
kubectl describe pvc my-pvc
# Causes:

# 1. "no persistent volumes available for this claim and no storage class is set"
#    → Définir storageClassName ou créer un PV

# 2. "storageclass.storage.k8s.io 'fast-ssd' not found"
#    → Vérifier le nom de la StorageClass
kubectl get storageclass

# 3. "waiting for first consumer to be created before binding"
#    → volumeBindingMode: WaitForFirstConsumer → normal, PV créé quand pod schedulé

# 4. Access mode incompatible
kubectl get pv -o custom-columns=NAME:.metadata.name,ACCESS:.spec.accessModes,CAPACITY:.spec.capacity.storage
```

### RBAC: permission refusée

```bash
# Erreur typique: "Error from server (Forbidden): pods is forbidden:
# User 'system:serviceaccount:default:my-sa' cannot list resource 'pods'..."

# Vérifier les permissions
kubectl auth can-i list pods \
  --as=system:serviceaccount:default:my-sa \
  --namespace=production

kubectl auth can-i --list \
  --as=system:serviceaccount:default:my-sa \
  --namespace=production

# Voir les bindings existants
kubectl get rolebindings,clusterrolebindings -A \
  -o custom-columns='KIND:.kind,NAMESPACE:.metadata.namespace,NAME:.metadata.name,SERVICEACCOUNTS:.subjects[?(@.kind=="ServiceAccount")].name'
```

---

## Commandes avancées (jsonpath, custom-columns)

```bash
# JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
kubectl get node worker-1 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Custom columns
kubectl get pods -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName,IP:.status.podIP'

kubectl get nodes -o custom-columns=\
'NAME:.metadata.name,STATUS:.status.conditions[-1].type,VERSION:.status.nodeInfo.kubeletVersion,CPU:.status.capacity.cpu,MEM:.status.capacity.memory'

# Trier
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime

# Filtrer par champ
kubectl get pods --field-selector=status.phase=Running
kubectl get pods --field-selector=spec.nodeName=worker-1
kubectl get pods --field-selector=metadata.namespace!=kube-system

# Watching
kubectl get pods -w                   # Watch mode
watch kubectl get pods                # Alternative avec watch

# Labels
kubectl get pods -l app=my-app
kubectl get pods -l 'app in (frontend,backend)'
kubectl get pods -l app!=my-app
kubectl get pods --show-labels
kubectl label pod my-pod env=debug    # Ajouter un label
kubectl label pod my-pod env-          # Supprimer un label

# Annotations
kubectl annotate pod my-pod note="debug pod"
kubectl annotate pod my-pod note-     # Supprimer

# Patch
kubectl patch deployment my-app -p '{"spec":{"replicas":5}}'
kubectl patch pod my-pod --type='json' \
  -p='[{"op":"replace","path":"/spec/containers/0/image","value":"nginx:1.26"}]'

# Force delete (pod bloqué en Terminating)
kubectl delete pod my-pod --grace-period=0 --force

# Port-forward (debug local)
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward deployment/my-app 8080:80
kubectl port-forward service/my-service 8080:80

# Proxy API
kubectl proxy --port=8001
# Accès: http://localhost:8001/api/v1/namespaces/default/pods

# Copier des fichiers
kubectl cp my-pod:/var/log/app.log ./app.log
kubectl cp ./config.yaml my-pod:/etc/config/config.yaml
```

---

## Vérification de la santé du cluster

```bash
# État général
kubectl cluster-info
kubectl get componentstatuses       # Déprécié mais utile
kubectl get nodes

# Vérifier l'API server
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /livez

# Vérifier les composants système
kubectl get pods -n kube-system
kubectl logs -n kube-system kube-apiserver-master-1
kubectl logs -n kube-system kube-scheduler-master-1
kubectl logs -n kube-system kube-controller-manager-master-1

# Métriques
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Audit des ressources
kubectl get all -A | grep -v Running | grep -v Completed
```
