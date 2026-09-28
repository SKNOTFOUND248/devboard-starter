# Helm Guide — Installation, Commands & DevBoard Chart

This guide covers:
1. Installing Helm
2. Popular/useful Helm commands
3. A quick example: deploying nginx with Helm
4. Converting the DevBoard `k8s/` (kind) manifests into a Helm chart

---

## 1. Installing Helm

### macOS (Homebrew)
```bash
brew install helm
```

### Linux (script install)
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Linux (apt - Debian/Ubuntu)
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Windows (Chocolatey)
```powershell
choco install kubernetes-helm
```

### Verify installation
```bash
helm version
```

---

## 2. Popular / Useful Helm Commands

### Repository management
```bash
helm repo add <name> <url>          # add a chart repo, e.g. helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update                    # refresh local repo cache
helm repo list                      # list added repos
helm search repo <keyword>          # search charts in added repos
helm search hub <keyword>           # search Artifact Hub
```

### Installing & upgrading releases
```bash
helm install <release-name> <chart> -n <namespace> --create-namespace
helm install <release-name> <chart> -f custom-values.yaml
helm install <release-name> <chart> --set key=value --set key2.subkey=value2
helm upgrade <release-name> <chart>                     # upgrade existing release
helm upgrade --install <release-name> <chart>           # install if missing, upgrade if exists
helm upgrade <release-name> <chart> -f values.yaml --atomic   # rollback automatically if upgrade fails
```

### Inspecting releases
```bash
helm list -A                        # list all releases across namespaces
helm list -n <namespace>            # list releases in a namespace
helm status <release-name>          # show status of a release
helm get values <release-name>      # show values used for a release
helm get manifest <release-name>    # show rendered k8s manifests for a release
helm get all <release-name>         # show everything (values, manifest, hooks, notes)
helm history <release-name>         # show revision history
```

### Rollback & uninstall
```bash
helm rollback <release-name> <revision>   # roll back to a previous revision
helm uninstall <release-name>             # delete a release
helm uninstall <release-name> --keep-history  # delete but keep revision history
```

### Chart development
```bash
helm create <chart-name>            # scaffold a new chart
helm lint <chart-path>              # validate chart syntax/best practices
helm template <chart-path>          # render templates locally (no cluster needed)
helm template <chart-path> -f values.yaml   # render with custom values
helm package <chart-path>           # package chart into a .tgz
helm dependency update <chart-path> # fetch subchart dependencies (from Chart.yaml)
helm show values <chart>            # print a chart's default values.yaml
helm show chart <chart>             # print Chart.yaml metadata
```

### Debugging
```bash
helm install <release-name> <chart> --dry-run --debug   # simulate install, print rendered manifests
helm upgrade <release-name> <chart> --dry-run --debug
```

---

## 3. Example: Deploying nginx with Helm

Using Bitnami's popular nginx chart:

```bash
# 1. Add the Bitnami repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. See default values (optional, good for customizing)
helm show values bitnami/nginx > nginx-values.yaml

# 3. Install nginx into its own namespace
helm install my-nginx bitnami/nginx \
  --namespace nginx-demo --create-namespace \
  --set service.type=NodePort

# 4. Check the release
helm status my-nginx -n nginx-demo
kubectl get all -n nginx-demo

# 5. Upgrade (e.g., scale replicas)
helm upgrade my-nginx bitnami/nginx -n nginx-demo --set replicaCount=2

# 6. Uninstall when done
helm uninstall my-nginx -n nginx-demo
```

This is the fastest way to see Helm's value: one command installs a Deployment, Service, and supporting resources that would otherwise be several YAML files.

---

## 4. Helm Chart for DevBoard (kind manifests)

The `k8s/` folder (ignoring `k8s/eks`) contains plain manifests for the DevBoard app on **kind**:

| File | Resource |
|---|---|
| `01-namespace.yml` | Namespace `devboard-ns` |
| `02-frontend-pod.yml` | Standalone frontend Pod (not used once Deployment exists — superseded by `03`) |
| `03-frontend-deployment.yml` | Frontend Deployment |
| `04-frontend-service.yml` | Frontend Service (NodePort 30001) |
| `05-backend-deployment.yml` | Backend Deployment (env from ConfigMap + Secret) |
| `06-secrets.yml` | Secret with `POSTGRES_PASSWORD` |
| `07-configmap.yml` | ConfigMap with `POSTGRES_USER`, `POSTGRES_DB`, `PORT` |
| `08-postgres.yml` | Postgres StatefulSet + PVC template |
| `09-postgres-service.yml` | Headless Postgres Service |
| `10-postgres-pv.yml` | PersistentVolume (hostPath, for kind) |
| `11-postgres-init.yml` | ConfigMap with init SQL (schema + seed) |
| `12-backend-service.yml` | Backend Service |

Below is a Helm chart, `devboard-chart/`, that templatizes these so the app can be installed/upgraded with one command instead of `kubectl apply -f k8s/`.

### 4.1 Chart layout

```
devboard-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── namespace.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── postgres-init-configmap.yaml
    ├── postgres-pv.yaml
    ├── postgres-statefulset.yaml
    ├── postgres-service.yaml
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── frontend-deployment.yaml
    └── frontend-service.yaml
```

### 4.2 `Chart.yaml`
```yaml
apiVersion: v2
name: devboard
description: Helm chart for DevBoard (frontend + backend + postgres) on kind
type: application
version: 0.1.0
appVersion: "1.0.0"
```

### 4.3 `values.yaml`
```yaml
namespace: devboard-ns

frontend:
  image: trainwithshubham/devboard-frontend:latest
  replicas: 1
  containerPort: 4173
  service:
    type: NodePort
    port: 80
    nodePort: 30001
  resources:
    requests:
      cpu: 10m
      memory: 64Mi
    limits:
      cpu: 250m
      memory: 128Mi

backend:
  image: trainwithshubham/devboard-backend:latest
  replicas: 1
  containerPort: 8080
  service:
    port: 8080
  resources:
    requests:
      cpu: 10m
      memory: 64Mi
    limits:
      cpu: 250m
      memory: 128Mi

postgres:
  image: postgres:16-alpine
  storage: 0.5Gi
  pvHostPath: /mnt/pg-data
  pvStorage: 1Gi
  storageClassName: manual

config:
  POSTGRES_USER: devboard
  POSTGRES_DB: devboard
  PORT: "8080"

secrets:
  # base64 for "devboard" — replace for real environments, this is a demo password
  POSTGRES_PASSWORD: ZGV2Ym9hcmQ=
```

### 4.4 `templates/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.namespace }}
```

### 4.5 `templates/configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: devboard-configmap
  namespace: {{ .Values.namespace }}
data:
  POSTGRES_USER: {{ .Values.config.POSTGRES_USER | quote }}
  POSTGRES_DB: {{ .Values.config.POSTGRES_DB | quote }}
  PORT: {{ .Values.config.PORT | quote }}
```

### 4.6 `templates/secret.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: devboard-secrets
  namespace: {{ .Values.namespace }}
data:
  POSTGRES_PASSWORD: {{ .Values.secrets.POSTGRES_PASSWORD }}
```

### 4.7 `templates/postgres-init-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
  namespace: {{ .Values.namespace }}
data:
  01_schema.sql: |
{{ .Files.Get "files/01_schema.sql" | indent 4 }}
  02_seed.sql: |
{{ .Files.Get "files/02_seed.sql" | indent 4 }}
```
> Copy the SQL bodies from `k8s/11-postgres-init.yml` into `devboard-chart/files/01_schema.sql` and `files/02_seed.sql` so Helm can load them with `.Files.Get`.

### 4.8 `templates/postgres-pv.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: {{ .Values.postgres.pvStorage }}
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: {{ .Values.postgres.storageClassName }}
  hostPath:
    path: {{ .Values.postgres.pvHostPath | quote }}
    type: DirectoryOrCreate
```
> `type: DirectoryOrCreate` matters: without it, kubelet errors
> `stat <path>: no such file or directory` instead of creating the directory
> — found by actually installing this chart on a kind cluster.

### 4.9 `templates/postgres-statefulset.yaml`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: {{ .Values.namespace }}
  labels:
    app: devboard-postgres
spec:
  replicas: 1
  serviceName: postgres
  selector:
    matchLabels:
      app: devboard-postgres
  template:
    metadata:
      labels:
        app: devboard-postgres
    spec:
      containers:
        - name: postgres
          image: {{ .Values.postgres.image }}
          ports:
            - containerPort: 5432
              name: web
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
              subPath: pgdata
            - name: init
              mountPath: /docker-entrypoint-initdb.d
          env:
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: devboard-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: POSTGRES_DB
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"
      volumes:
        - name: init
          configMap:
            name: postgres-init
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: {{ .Values.postgres.storageClassName | quote }}
        resources:
          requests:
            storage: {{ .Values.postgres.storage }}
```
> Note: added `subPath: pgdata` so Postgres's `PGDATA` doesn't sit at the volume root (avoids the lost+found initdb issue fixed in the `k8s/eks` manifests).

### 4.10 `templates/postgres-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: {{ .Values.namespace }}
spec:
  clusterIP: None
  selector:
    app: devboard-postgres
  ports:
    - protocol: TCP
      port: 5432
      targetPort: 5432
```

### 4.11 `templates/backend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: {{ .Values.namespace }}
  labels:
    app: devboard-be
spec:
  replicas: {{ .Values.backend.replicas }}
  selector:
    matchLabels:
      app: devboard-be
  template:
    metadata:
      labels:
        app: devboard-be
    spec:
      containers:
        - name: devboard-backend
          image: {{ .Values.backend.image }}
          ports:
            - containerPort: {{ .Values.backend.containerPort }}
          env:
            - name: PORT
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: PORT
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: devboard-secrets
                  key: POSTGRES_PASSWORD
            - name: POSTGRES_USER
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: POSTGRES_USER
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: devboard-configmap
                  key: POSTGRES_DB
            - name: POSTGRES_URL
              value: postgres://$(POSTGRES_USER):$(POSTGRES_PASSWORD)@postgres:5432/$(POSTGRES_DB)?sslmode=disable
          resources:
{{ toYaml .Values.backend.resources | indent 12 }}
```

### 4.12 `templates/backend-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: devboard-be
  ports:
    - protocol: TCP
      port: {{ .Values.backend.service.port }}
      targetPort: {{ .Values.backend.containerPort }}
```

### 4.13 `templates/frontend-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: {{ .Values.namespace }}
  labels:
    app: devboard-fe
spec:
  replicas: {{ .Values.frontend.replicas }}
  selector:
    matchLabels:
      app: devboard-fe
  template:
    metadata:
      labels:
        app: devboard-fe
    spec:
      containers:
        - name: devboard-frontend
          image: {{ .Values.frontend.image }}
          ports:
            - containerPort: {{ .Values.frontend.containerPort }}
          resources:
{{ toYaml .Values.frontend.resources | indent 12 }}
```

### 4.14 `templates/frontend-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: {{ .Values.namespace }}
spec:
  selector:
    app: devboard-fe
  ports:
    - protocol: TCP
      port: {{ .Values.frontend.service.port }}
      targetPort: {{ .Values.frontend.containerPort }}
      nodePort: {{ .Values.frontend.service.nodePort }}
  type: {{ .Values.frontend.service.type }}
```

### 4.15 Installing the chart on kind

```bash
# from repo root
helm lint devboard-chart
helm template devboard-chart               # sanity-check rendered manifests

helm install devboard devboard-chart --create-namespace -n devboard-ns

# check
kubectl get all -n devboard-ns

# access frontend (NodePort 30001) -- if kind cluster has extraPortMappings, else port-forward:
kubectl port-forward -n devboard-ns svc/frontend-service 8080:80

# upgrade (e.g. bump replicas)
helm upgrade devboard devboard-chart -n devboard-ns --set frontend.replicas=2

# uninstall
helm uninstall devboard -n devboard-ns
```

**Verified on a real kind cluster (`v1.36.1`, matching `kind/kind-config.yaml`):**
`helm lint` and `helm template` both pass cleanly, and a real `helm install`
correctly creates the namespace, ConfigMap, Secret, and the postgres
StatefulSet + PVC + static PV (bound via `storageClassName: manual`) — the
postgres pod reaches `1/1 Running` and the seeded schema/data load
correctly (`select count(*) from tasks` → 10, `projects` → 2). Service specs
(backend `ClusterIP:8080`, frontend `NodePort 80:30001`, postgres headless)
and the backend Deployment's env resolution (`configMapKeyRef`/
`secretKeyRef`/`$(POSTGRES_USER)` interpolation) were confirmed structurally
correct against the live API server.

> **Known gap, not a chart bug:** `trainwithshubham/devboard-frontend` and
> `devboard-backend` are currently published as **amd64-only** images (no
> arm64 manifest). On an Apple Silicon Mac, a local kind cluster runs
> arm64 nodes, so pulling either image fails with `no match for platform
> in manifest: not found` — regardless of whether you're using this chart
> or the plain `k8s/` manifests. Fix belongs in the image build pipeline
> (multi-arch `docker buildx build --platform linux/amd64,linux/arm64`),
> not in this chart. On an amd64 machine (or EKS, which uses amd64 nodes by
> default), this doesn't apply.

### 4.16 Why this is better than raw `kubectl apply -f k8s/`
- **Single command install/upgrade/rollback** (`helm install/upgrade/rollback`) instead of applying 12 files in order.
- **Configurable via `values.yaml`** — change image tags, replica counts, resource limits, or storage size without editing manifests.
- **Versioned releases** — `helm history devboard` tracks every change; `helm rollback` undoes a bad upgrade instantly.
- **Environment-specific overrides** — a `values-eks.yaml` could later target the EKS setup (in `k8s/eks`) using the same chart with different values, instead of maintaining a fully separate manifest set.
