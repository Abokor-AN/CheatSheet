# ============================================================
# KUSTOMIZE - CHEATSHEET COMPLET
# ============================================================
# Outil natif kubectl pour personnaliser des configs K8s sans templates.
# Intégré dans kubectl: kubectl apply -k ./
# Principe: base + overlays = configuration finale

## Structure de projet recommandée

```
my-app/
├── base/                           # Configuration commune
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── serviceaccount.yaml
│
└── overlays/                       # Personnalisations par env
    ├── development/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── deployment-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── deployment-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── hpa.yaml
        └── patches/
            ├── deployment-patch.yaml
            └── resources-patch.yaml
```

---

## base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# ─── RESSOURCES ───────────────────────────────────────────
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - serviceaccount.yaml
  # URL externe (référence à une release GitHub)
  - https://github.com/myorg/myapp/config/base/?ref=v1.0.0
  # Autre dossier Kustomize
  - ../common/

# ─── NAMESPACE ────────────────────────────────────────────
namespace: my-app                     # Appliqué à toutes les ressources

# ─── PREFIXES/SUFFIXES ────────────────────────────────────
namePrefix: "myteam-"                 # Préfixe sur tous les noms
nameSuffix: "-v2"

# ─── LABELS COMMUNS ───────────────────────────────────────
commonLabels:
  app.kubernetes.io/name: my-app
  app.kubernetes.io/managed-by: kustomize

# ─── ANNOTATIONS COMMUNES ─────────────────────────────────
commonAnnotations:
  owner: "team-backend"

# ─── IMAGES ───────────────────────────────────────────────
images:
  - name: my-app                      # Nom de l'image dans le YAML
    newName: registry.example.com/my-app  # Nouvelle image
    newTag: "1.2.3"                   # Nouveau tag
  - name: nginx
    newTag: "1.25-alpine"
  - name: my-sidecar
    digest: sha256:abc123...          # Immutable digest

# ─── CONFIGMAPS GÉNÉRÉS ───────────────────────────────────
configMapGenerator:
  - name: app-config
    literals:
      - LOG_LEVEL=info
      - APP_PORT=8080
    files:
      - configs/app.properties        # clé = nom du fichier
      - app.conf=configs/application.conf  # clé personnalisée
    envs:
      - app.env                       # Fichier .env (KEY=VALUE)
    options:
      disableNameSuffixHash: true     # Désactiver le hash dans le nom
      labels:
        app: my-app
      annotations:
        note: "generated"

# ─── SECRETS GÉNÉRÉS ──────────────────────────────────────
secretGenerator:
  - name: app-secrets
    literals:
      - password=mysecret
    files:
      - secret.txt
    envs:
      - secrets.env
    type: Opaque                      # Opaque | kubernetes.io/tls | etc.
    options:
      disableNameSuffixHash: false

# ─── PATCHES ──────────────────────────────────────────────
patches:
  # Strategic Merge Patch (fichier)
  - path: patches/deployment-patch.yaml

  # Strategic Merge Patch (inline)
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
      spec:
        replicas: 1

  # JSON Patch (RFC 6902)
  - patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: DEBUG
          value: "true"
    target:
      kind: Deployment
      name: my-deployment

  # Avec target sélecteur
  - path: patches/add-label.yaml
    target:
      kind: Deployment               # Appliquer à tous les Deployments
      labelSelector: "app=my-app"
      annotationSelector: "env=production"

# ─── PATCH STRATEGIQUE PAR MERGE ──────────────────────────
patchesStrategicMerge:               # Déprécié - utiliser patches à la place
  - deployment-patch.yaml

# ─── PATCH JSON6902 ───────────────────────────────────────
patchesJson6902:                     # Déprécié - utiliser patches à la place
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-deployment
    path: patches/json-patch.yaml

# ─── REMPLACEMENTS ────────────────────────────────────────
replacements:
  - source:
      kind: ConfigMap
      name: app-config
      fieldPath: data.APP_PORT
    targets:
      - select:
          kind: Service
          name: my-service
        fieldPaths:
          - spec.ports.0.port

# ─── COMPOSANTS ───────────────────────────────────────────
components:
  - ../components/monitoring          # Réutilisable entre overlays

# ─── GÉNÉRATEURS PERSONNALISÉS ────────────────────────────
generators:
  - my-generator-plugin.yaml

# ─── TRANSFORMERS PERSONNALISÉS ───────────────────────────
transformers:
  - my-transformer.yaml

# ─── VALIDATORS ───────────────────────────────────────────
validators:
  - my-validator.yaml

# ─── OPENAPI ──────────────────────────────────────────────
openapi:
  path: custom-schema.json           # Schéma OpenAPI pour les patches
```

---

## overlays/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Référence à la base
resources:
  - ../../base
  - hpa.yaml                         # Ressource spécifique à la prod

namespace: production

# Override d'images pour la prod
images:
  - name: my-app
    newTag: "2.0.0"                  # Tag de prod

# Patches spécifiques à la prod
patches:
  - path: patches/deployment-patch.yaml
  - path: patches/resources-patch.yaml

# ConfigMap spécifique à la prod
configMapGenerator:
  - name: app-config
    behavior: merge                  # merge | replace | create (défaut)
    literals:
      - LOG_LEVEL=warning
      - ENVIRONMENT=production

# Labels additionnels
commonLabels:
  environment: production
```

---

## overlays/development/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

namespace: development

images:
  - name: my-app
    newTag: "latest"

patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: my-deployment
      spec:
        replicas: 1
        template:
          spec:
            containers:
              - name: app
                resources:
                  limits:
                    cpu: "200m"
                    memory: "128Mi"

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - LOG_LEVEL=debug
      - ENVIRONMENT=development
```

---

## Exemples de patches

### Strategic Merge Patch (patches/deployment-patch.yaml)

```yaml
# Modifie seulement ce qui est spécifié, merge intelligent
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: app
          resources:
            limits:
              cpu: "2"
              memory: "2Gi"
            requests:
              cpu: "500m"
              memory: "512Mi"
          env:
            - name: NEW_VAR
              value: "new-value"
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: kubernetes.io/hostname
```

### Supprimer un élément (Strategic Merge)

```yaml
# Supprimer un container ou volume avec $patch: delete
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  template:
    spec:
      containers:
        - name: sidecar
          $patch: delete             # Supprime ce container
```

### JSON Patch (patches/json-patch.yaml)

```yaml
# RFC 6902 - opérations précises
- op: replace
  path: /spec/replicas
  value: 5

- op: add
  path: /spec/template/spec/containers/0/env/-
  value:
    name: NEW_ENV
    value: "value"

- op: remove
  path: /spec/template/spec/containers/0/livenessProbe

- op: copy
  from: /spec/template/spec/containers/0/resources
  path: /spec/template/spec/initContainers/0/resources

- op: move
  from: /spec/template/metadata/labels/old-label
  path: /spec/template/metadata/labels/new-label

- op: test
  path: /spec/replicas
  value: 3                           # Valide avant d'appliquer
```

---

## Composants Kustomize (réutilisables)

```
components/
└── monitoring/
    ├── kustomization.yaml
    └── servicemonitor.yaml
```

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component                      # Pas Kustomization!

resources:
  - servicemonitor.yaml

patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: not-important
      spec:
        template:
          metadata:
            annotations:
              prometheus.io/scrape: "true"
              prometheus.io/port: "9090"
    target:
      kind: Deployment
```

---

## Commandes

```bash
# ─── BUILD & APPLY ──────────────────────────────────────────
kubectl kustomize ./                  # Afficher le YAML généré
kubectl kustomize overlays/production
kubectl apply -k overlays/production
kubectl apply -k overlays/production --dry-run=client
kubectl apply -k overlays/production --dry-run=server

# ─── DIFF ───────────────────────────────────────────────────
kubectl diff -k overlays/production  # Différences avec le cluster actuel

# ─── DELETE ─────────────────────────────────────────────────
kubectl delete -k overlays/production

# ─── KUSTOMIZE CLI (standalone) ─────────────────────────────
kustomize build overlays/production
kustomize build overlays/production | kubectl apply -f -
kustomize build overlays/production > rendered.yaml

# Avec Helm (via HelmChartInflationGenerator)
kustomize build --enable-helm overlays/production

# ─── EDIT ───────────────────────────────────────────────────
kustomize edit set image my-app=registry.example.com/my-app:2.0.0
kustomize edit set namespace production
kustomize edit set replicas my-deployment=5
kustomize edit add resource hpa.yaml
kustomize edit add patch patch.yaml
kustomize edit add label environment:production
kustomize edit add annotation owner:team-backend
kustomize edit add configmap app-config --from-literal=KEY=VALUE

# ─── INTÉGRATION HELM ───────────────────────────────────────
# Dans kustomization.yaml avec helmCharts:
helmCharts:
  - name: nginx
    repo: https://charts.bitnami.com/bitnami
    version: "15.0.0"
    releaseName: my-nginx
    namespace: default
    valuesFile: values.yaml
    additionalValuesFiles:
      - values-prod.yaml
```

---

## Kustomize vs Helm

| | Kustomize | Helm |
|---|---|---|
| **Apprentissage** | Simple (YAML pur) | Plus complexe (templates Go) |
| **Templating** | Non (patches) | Oui (Go templates) |
| **Variables** | Non natif | values.yaml |
| **Packages** | Non | Oui (Artifact Hub) |
| **Rollback** | Manuel (git) | `helm rollback` |
| **Cycle de vie** | kubectl | helm install/upgrade |
| **Intégration kubectl** | Natif | Plugin |
| **GitOps** | Idéal | Bien supporté |
| **Best for** | Configurations internes | Apps packagées/distribuées |
