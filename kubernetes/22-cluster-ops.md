# ============================================================
# OPÉRATIONS CLUSTER - CHEATSHEET COMPLET
# ============================================================
# kubeadm, etcd backup/restore, upgrade, certificats TLS

## kubeadm - Installation et gestion de cluster

### Pré-requis (sur chaque nœud)

```bash
# Désactiver swap (requis par K8s)
swapoff -a
sed -i '/swap/d' /etc/fstab

# Charger les modules kernel
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Paramètres sysctl
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system

# Installer containerd
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
# Modifier: SystemdCgroup = true
systemctl restart containerd

# Installer kubeadm, kubelet, kubectl
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl   # Empêcher upgrade automatique
```

### Initialiser le cluster (control plane)

```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \   # Pour Flannel
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version=v1.29.0 \
  --control-plane-endpoint=my-lb:6443 \ # Pour HA (multi control-plane)
  --upload-certs \                       # Pour ajouter d'autres control planes
  --cri-socket=unix:///run/containerd/containerd.sock \
  --image-repository=registry.k8s.io

# Config kubectl
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Installer un CNI (ex: Calico)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# ou Flannel:
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
# ou Cilium:
helm install cilium cilium/cilium --namespace kube-system
```

### Rejoindre un nœud worker

```bash
# Récupérer la commande join (générée par kubeadm init)
kubeadm token create --print-join-command

# Rejoindre en tant que worker
kubeadm join my-lb:6443 \
  --token abcdef.1234567890abcdef \
  --discovery-token-ca-cert-hash sha256:abc123...

# Rejoindre en tant que control plane (HA)
kubeadm join my-lb:6443 \
  --token abcdef.1234567890abcdef \
  --discovery-token-ca-cert-hash sha256:abc123... \
  --control-plane \
  --certificate-key <cert-key>

# Si token expiré (24h), en générer un nouveau
kubeadm token create
kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
  openssl rsa -pubin -outform der 2>/dev/null | \
  openssl dgst -sha256 -hex | sed 's/^.* //'  # CA cert hash
```

### Configuration kubeadm (fichier)

```yaml
# kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: "1.29.0"
controlPlaneEndpoint: "my-lb:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
etcd:
  local:
    dataDir: /var/lib/etcd
    extraArgs:
      quota-backend-bytes: "8589934592"   # 8GB
apiServer:
  certSANs:
    - "my-lb.example.com"
    - "203.0.113.100"
  extraArgs:
    audit-log-path: /var/log/kubernetes/audit.log
    audit-log-maxage: "30"
    encryption-provider-config: /etc/kubernetes/encryption-config.yaml
  extraVolumes:
    - name: encryption-config
      hostPath: /etc/kubernetes/encryption-config.yaml
      mountPath: /etc/kubernetes/encryption-config.yaml
      readOnly: true
controllerManager:
  extraArgs:
    bind-address: "0.0.0.0"
scheduler:
  extraArgs:
    bind-address: "0.0.0.0"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
  taints: []                          # Pas de taint sur le control plane
  kubeletExtraArgs:
    node-labels: "node-type=control-plane"
```

```bash
kubeadm init --config kubeadm-config.yaml
```

---

## Upgrade du cluster

### Upgrade du control plane

```bash
# 1. Vérifier les versions disponibles
apt-cache madison kubeadm
kubeadm upgrade plan

# 2. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.30.0-1.1
apt-mark hold kubeadm

# 3. Planifier et appliquer l'upgrade
kubeadm upgrade plan v1.30.0
kubeadm upgrade apply v1.30.0         # Premier control plane
# Sur les autres control planes:
kubeadm upgrade node

# 4. Drain du nœud
kubectl drain master-1 \
  --ignore-daemonsets \
  --delete-emptydir-data

# 5. Upgrade kubelet et kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
apt-mark hold kubelet kubectl

# 6. Redémarrer kubelet
systemctl daemon-reload
systemctl restart kubelet

# 7. Uncordon
kubectl uncordon master-1

# 8. Vérifier
kubectl get nodes
```

### Upgrade des nœuds workers

```bash
# Répéter pour chaque worker:

# 1. Cordon + drain (depuis le control plane)
kubectl cordon worker-1
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# 2. Sur le worker:
apt-mark unhold kubeadm kubelet kubectl
apt-get install -y \
  kubeadm=1.30.0-1.1 \
  kubelet=1.30.0-1.1 \
  kubectl=1.30.0-1.1
apt-mark hold kubeadm kubelet kubectl

kubeadm upgrade node

systemctl daemon-reload
systemctl restart kubelet

# 3. Uncordon (depuis le control plane)
kubectl uncordon worker-1
```

---

## etcd - Backup et Restore

### Backup etcd

```bash
# Variables
ETCD_ENDPOINTS="https://127.0.0.1:2379"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"
BACKUP_FILE="/backup/etcd-$(date +%Y%m%d_%H%M%S).db"

# Créer le snapshot
ETCDCTL_API=3 etcdctl snapshot save $BACKUP_FILE \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Vérifier le snapshot
ETCDCTL_API=3 etcdctl snapshot status $BACKUP_FILE \
  --write-out=table

# Ou avec le pod etcd (cluster kubeadm)
kubectl exec -n kube-system etcd-master-1 -- \
  etcdctl snapshot save /var/lib/etcd/snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Copier le snapshot hors du pod
kubectl cp kube-system/etcd-master-1:/var/lib/etcd/snapshot.db ./snapshot.db
```

### Restore etcd

```bash
# ATTENTION: Arrêter le kube-apiserver avant le restore!

# 1. Arrêter l'API server (supprimer le manifest static pod)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# Attendre que les containers s'arrêtent
crictl ps | grep etcd  # Doit être vide

# 2. Restaurer le snapshot
ETCDCTL_API=3 etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-restore \
  --name=master-1 \
  --initial-cluster="master-1=https://127.0.0.1:2380" \
  --initial-cluster-token=etcd-cluster-token \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# 3. Remplacer le répertoire etcd
mv /var/lib/etcd /var/lib/etcd.bak
mv /var/lib/etcd-restore /var/lib/etcd
chown -R etcd:etcd /var/lib/etcd  # Si nécessaire

# 4. Remettre les manifests
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Attendre que le cluster soit disponible
kubectl get nodes
```

### Opérations etcd courantes

```bash
# Statut du cluster etcd
ETCDCTL_API=3 etcdctl member list \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY \
  --write-out=table

# Santé du cluster
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Lister les clés
ETCDCTL_API=3 etcdctl get / --prefix --keys-only \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Voir une valeur
ETCDCTL_API=3 etcdctl get /registry/pods/default/my-pod \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Compactage et défragmentation (maintenance)
ETCDCTL_API=3 etcdctl defrag \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY
```

---

## Certificats TLS

### Vérifier l'expiration des certificats

```bash
# Voir tous les certificats kubeadm
kubeadm certs check-expiration

# Exemple de sortie:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME
# admin.conf                 Jan 01, 2026 00:00 UTC   364d
# apiserver                  Jan 01, 2026 00:00 UTC   364d
# apiserver-etcd-client      Jan 01, 2026 00:00 UTC   364d
# apiserver-kubelet-client   Jan 01, 2026 00:00 UTC   364d
# controller-manager.conf    Jan 01, 2026 00:00 UTC   364d
# etcd-healthcheck-client    Jan 01, 2026 00:00 UTC   364d
# etcd-peer                  Jan 01, 2026 00:00 UTC   364d
# etcd-server                Jan 01, 2026 00:00 UTC   364d
# front-proxy-client         Jan 01, 2026 00:00 UTC   364d
# scheduler.conf             Jan 01, 2026 00:00 UTC   364d

# Voir un certificat spécifique
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 Validity
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates
```

### Renouveler les certificats

```bash
# Renouveler tous les certificats (1 an)
kubeadm certs renew all

# Renouveler un certificat spécifique
kubeadm certs renew apiserver
kubeadm certs renew apiserver-etcd-client
kubeadm certs renew apiserver-kubelet-client
kubeadm certs renew admin.conf

# Après renouvellement: redémarrer les composants
systemctl restart kubelet
# Les static pods redémarrent automatiquement

# Copier le nouveau kubeconfig
cp /etc/kubernetes/admin.conf ~/.kube/config
```

### Créer un certificat utilisateur

```bash
# 1. Générer une clé privée et CSR
openssl genrsa -out alice.key 2048
openssl req -new -key alice.key \
  -out alice.csr \
  -subj "/CN=alice/O=dev-team"    # CN = username, O = group

# 2. Créer un CertificateSigningRequest K8s
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice-csr
spec:
  request: $(base64 -w 0 < alice.csr)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400           # 24h
  usages:
    - client auth
EOF

# 3. Approuver le CSR
kubectl get csr
kubectl certificate approve alice-csr
# kubectl certificate deny alice-csr  # Refuser

# 4. Récupérer le certificat signé
kubectl get csr alice-csr \
  -o jsonpath='{.status.certificate}' | base64 -d > alice.crt

# 5. Créer le kubeconfig
kubectl config set-credentials alice \
  --client-certificate=alice.crt \
  --client-key=alice.key

kubectl config set-context alice-context \
  --cluster=my-cluster \
  --namespace=my-namespace \
  --user=alice

# 6. Donner les droits (RBAC)
kubectl create rolebinding alice-binding \
  --clusterrole=edit \
  --user=alice \
  --namespace=my-namespace
```

---

## Cluster Autoscaler

```bash
# Voir l'état du Cluster Autoscaler
kubectl -n kube-system get pods -l app=cluster-autoscaler
kubectl -n kube-system logs -l app=cluster-autoscaler -f

# Annotations sur un nœud pour contrôler l'autoscaler
kubectl annotate node worker-1 \
  cluster-autoscaler.kubernetes.io/scale-down-disabled="true"  # Protéger le nœud

# Voir pourquoi un pod est non-schedulable
kubectl get events -n my-ns --field-selector reason=FailedScheduling

# Désactiver temporairement l'autoscaler
kubectl -n kube-system scale deployment cluster-autoscaler --replicas=0
```

---

## Velero - Backup/Restore du cluster

```bash
# Installation
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-backup-bucket \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Backup
velero backup create my-backup \
  --include-namespaces production \
  --exclude-resources secrets \
  --ttl 720h                         # 30 jours

velero backup create full-cluster-backup  # Tout le cluster
velero backup describe my-backup
velero backup logs my-backup

# Schedule (backup automatique)
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production

# Restore
velero restore create --from-backup my-backup
velero restore create my-restore \
  --from-backup my-backup \
  --include-namespaces production \
  --namespace-mappings production:production-restored

velero restore describe my-restore
velero restore logs my-restore

# Liste
velero backup get
velero schedule get
velero restore get
```
