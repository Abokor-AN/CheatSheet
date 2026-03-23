# ============================================================
# HELM - CHEATSHEET COMPLET
# ============================================================
# Gestionnaire de packages Kubernetes.
# Concepts: Chart, Release, Repository, Values, Templates

## Structure d'un Chart

```
my-chart/
├── Chart.yaml              # Métadonnées du chart
├── values.yaml             # Valeurs par défaut
├── values-prod.yaml        # Valeurs de surcharge (optionnel, convention)
├── charts/                 # Charts dépendants
├── templates/              # Templates Kubernetes
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── _helpers.tpl        # Helpers/partials (non rendus directement)
│   ├── NOTES.txt           # Notes post-installation
│   └── tests/
│       └── test-connection.yaml
├── crds/                   # CRDs (installés avant les templates)
└── .helmignore             # Fichiers à ignorer (comme .gitignore)
```

---

## Chart.yaml

```yaml
apiVersion: v2                        # v1 (Helm 2) | v2 (Helm 3)
name: my-chart
description: "Mon application"
type: application                     # application | library
version: "1.2.3"                      # Version du chart (SemVer)
appVersion: "2.0.0"                   # Version de l'application (informatif)

keywords: [web, backend, api]
home: https://example.com
sources:
  - https://github.com/myorg/myapp
maintainers:
  - name: John Doe
    email: john@example.com

icon: https://example.com/icon.png
deprecated: false

annotations:
  category: "Database"
  licenses: "Apache-2.0"
  artifacthub.io/changes: |
    - kind: added
      description: New feature X

# Dépendances
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled      # Activer/désactiver via values
    tags: [database]
    import-values:                     # Importer des values du sous-chart
      - data
  - name: redis
    version: ">=17.0.0"
    repository: "@bitnami"             # Alias de repo
    condition: redis.enabled
  - name: common                       # Chart library
    version: "^2.0.0"
    repository: "https://charts.bitnami.com/bitnami"
```

---

## values.yaml

```yaml
# Valeurs par défaut - toutes les options configurables

replicaCount: 2

image:
  repository: my-registry/my-app
  tag: ""                             # Override par appVersion si vide
  pullPolicy: IfNotPresent
  pullSecrets: []

nameOverride: ""                      # Override le nom du chart
fullnameOverride: ""                  # Override le nom complet

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext:
  fsGroup: 2000

securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5

nodeSelector: {}
tolerations: []
affinity: {}

# Dépendances
postgresql:
  enabled: true
  auth:
    username: myapp
    password: ""                      # Généré si vide
    database: myappdb
  primary:
    persistence:
      size: 10Gi

redis:
  enabled: false
  auth:
    enabled: true

# Config application
config:
  logLevel: info
  environment: production

# Secrets (ne jamais committer les vraies valeurs)
secrets:
  databasePassword: ""
  apiKey: ""
```

---

## Templates (templates/)

### deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-chart.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-chart.fullname" . }}-secrets
                  key: databasePassword
          {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### _helpers.tpl

```
{{/*
Expand the name of the chart.
*/}}
{{- define "my-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-chart.labels" -}}
helm.sh/chart: {{ include "my-chart.chart" . }}
{{ include "my-chart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Service account name
*/}}
{{- define "my-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## Fonctions de template Go/Sprig

```
# Contrôle de flux
{{- if .Values.ingress.enabled }}   # if (- = trim whitespace)
{{- else if eq .Values.service.type "NodePort" }}
{{- end }}

{{- range .Values.ingress.hosts }}  # boucle
  - host: {{ .host }}
{{- end }}

{{- with .Values.nodeSelector }}    # if not nil/empty
  nodeSelector:
    {{- toYaml . | nindent 8 }}
{{- end }}

# Fonctions utiles
{{ .Values.name | default "default-name" }}
{{ .Values.name | quote }}          # Ajoute guillemets: "value"
{{ .Values.name | upper }}
{{ .Values.name | lower }}
{{ .Values.name | trunc 63 }}
{{ .Values.name | trimSuffix "-" }}
{{ printf "%s-%s" .Release.Name .Chart.Name }}
{{ toYaml .Values.resources | nindent 12 }}
{{ toJson .Values.config }}
{{ include "my-chart.labels" . | nindent 4 }}  # Inclure un template défini
{{ sha256sum "content" }}           # Hash (pour forcer redéploiement)
{{ randAlphaNum 16 }}               # Mot de passe aléatoire
{{ b64enc "secret" }}               # Encoder base64
{{ b64dec "c2VjcmV0" }}             # Décoder base64
{{ now | date "2006-01-02" }}       # Date formatée
{{ required "image.tag is required" .Values.image.tag }}  # Valeur requise
{{ fail "Invalid value" }}          # Erreur manuelle

# Variables
{{- $fullname := include "my-chart.fullname" . -}}
name: {{ $fullname }}

# Scope
{{- range $key, $val := .Values.env }}
- name: {{ $key }}
  value: {{ $val | quote }}
{{- end }}
```

---

## Objets de contexte

```
.Release.Name          # Nom de la release (ex: my-release)
.Release.Namespace     # Namespace de déploiement
.Release.IsInstall     # true si première installation
.Release.IsUpgrade     # true si mise à jour
.Release.Revision      # Numéro de révision

.Chart.Name            # Nom du chart
.Chart.Version         # Version du chart
.Chart.AppVersion      # Version de l'app
.Chart.Description

.Values                # Contenu de values.yaml (+ overrides)
.Files.Get "file.txt"  # Lire un fichier du chart
.Files.Glob "configs/*"
.Capabilities.KubeVersion.Major   # Version K8s
.Capabilities.KubeVersion.Minor
.Capabilities.APIVersions.Has "apps/v1"
.Template.Name         # Chemin du template courant
```

---

## Commandes Helm CLI

```bash
# ─── REPOSITORIES ───────────────────────────────────────────
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add stable https://charts.helm.sh/stable
helm repo list
helm repo update                      # Refresh index
helm repo remove bitnami

# Recherche
helm search repo postgresql           # Dans les repos ajoutés
helm search hub nginx                 # Sur Artifact Hub (public)
helm show chart bitnami/postgresql
helm show values bitnami/postgresql
helm show readme bitnami/postgresql
helm show all bitnami/postgresql

# ─── INSTALLATION ───────────────────────────────────────────
helm install my-release my-chart/               # Depuis un dossier local
helm install my-release bitnami/nginx           # Depuis un repo
helm install my-release oci://registry/chart    # Depuis un registre OCI
helm install my-release my-chart.tgz            # Depuis une archive

helm install my-release bitnami/nginx \
  --namespace production \
  --create-namespace \
  --set image.tag=1.25 \
  --set replicaCount=3 \
  --values values-prod.yaml \
  --values values-secrets.yaml \
  --set-string config.debug="true" \
  --set-json 'annotations={"key":"value"}' \
  --timeout 5m \
  --wait \                            # Attendre que les pods soient prêts
  --wait-for-jobs \
  --atomic \                          # Rollback auto si échec
  --dry-run \                         # Simuler sans appliquer
  --debug \                           # Verbose
  --version "12.1.0"                  # Version spécifique

# ─── UPGRADE ────────────────────────────────────────────────
helm upgrade my-release bitnami/nginx \
  --namespace production \
  --values values-prod.yaml \
  --set image.tag=1.26 \
  --reuse-values \                    # Garder les values précédentes
  --reset-values \                    # Ignorer les values précédentes
  --atomic \
  --cleanup-on-fail \
  --timeout 5m \
  --wait

# Install ou upgrade en une commande
helm upgrade --install my-release bitnami/nginx \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml

# ─── ROLLBACK ───────────────────────────────────────────────
helm history my-release -n production          # Historique des révisions
helm rollback my-release 2 -n production       # Rollback vers révision 2
helm rollback my-release 0 -n production       # Rollback vers précédente

# ─── INSPECTION ─────────────────────────────────────────────
helm list                                       # Releases dans namespace courant
helm list -A                                    # Toutes les releases
helm list -n production
helm list --failed
helm list --pending
helm status my-release -n production
helm get values my-release -n production        # Values utilisées
helm get values my-release -n production --all # Toutes les values (avec défauts)
helm get manifest my-release -n production      # YAML généré
helm get hooks my-release -n production
helm get notes my-release -n production

# ─── SUPPRESSION ────────────────────────────────────────────
helm uninstall my-release -n production
helm uninstall my-release -n production --keep-history  # Garder l'historique

# ─── DÉVELOPPEMENT ──────────────────────────────────────────
helm create my-chart                    # Créer un chart depuis template
helm lint my-chart/                     # Vérifier le chart
helm lint my-chart/ -f values-prod.yaml
helm template my-release my-chart/     # Rendre les templates localement
helm template my-release my-chart/ \
  --values values-prod.yaml \
  --set image.tag=1.25 \
  --output-dir ./rendered \
  --namespace production
helm package my-chart/                 # Créer une archive .tgz
helm dependency update my-chart/       # Télécharger les dépendances
helm dependency list my-chart/
helm dependency build my-chart/

# ─── OCI REGISTRY ───────────────────────────────────────────
helm registry login registry.example.com \
  --username myuser \
  --password mypassword
helm push my-chart-1.0.0.tgz oci://registry.example.com/charts
helm pull oci://registry.example.com/charts/my-chart --version 1.0.0
helm install my-release oci://registry.example.com/charts/my-chart \
  --version 1.0.0

# ─── PLUGINS UTILES ─────────────────────────────────────────
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade my-release bitnami/nginx --values values-prod.yaml

helm plugin install https://github.com/jkroepke/helm-secrets
helm secrets upgrade my-release my-chart/ -f secrets.yaml

# ─── VARIABLES D'ENVIRONMENT ────────────────────────────────
HELM_NAMESPACE=production
HELM_DEBUG=true
KUBECONFIG=~/.kube/config
```

---

## Hooks

```yaml
# Dans un template:
annotations:
  "helm.sh/hook": pre-install          # pre-install | post-install
                                        # pre-upgrade | post-upgrade
                                        # pre-rollback | post-rollback
                                        # pre-delete | post-delete
                                        # test
  "helm.sh/hook-weight": "-5"          # Ordre d'exécution (plus petit = premier)
  "helm.sh/hook-delete-policy": hook-succeeded  # before-hook-creation | hook-succeeded | hook-failed
```

## Tests

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "my-chart.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "my-chart.fullname" . }}:{{ .Values.service.port }}']
```

```bash
helm test my-release -n production    # Exécuter les tests
```
