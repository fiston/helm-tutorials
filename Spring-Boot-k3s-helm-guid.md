# Spring Boot → k3s Deployment via Shared Helm Chart Templates

> Stack: Kotlin · Spring Boot · Docker · k3s · Helm · GitLab CI · Azure DevOps  
> Pattern: one shared `helm-charts` repo consumed by N microservice repos

---

## Overview

```
helm-charts-repo/          ← shared, versioned, reusable
  charts/
    springboot-app/        ← generic chart (one template fits all)

my-service-repo/           ← per-project
  src/
  helm/
    values.yaml            ← project-specific overrides
  .gitlab-ci.yml
  Dockerfile
```

---

## Part 1 — Create the Shared Helm Chart Template Repo

### 1.1 Scaffold the chart

```bash
helm create springboot-app
```

This generates boilerplate. Clean it up and replace with the structure below.

### 1.2 Final folder structure

```
helm-charts/
├── README.md
└── charts/
    └── springboot-app/
        ├── Chart.yaml
        ├── values.yaml                    ← defaults (safe fallbacks)
        └── templates/
            ├── _helpers.tpl
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml
            ├── configmap.yaml
            ├── secret.yaml
            ├── serviceaccount.yaml
            └── hpa.yaml
```

---

### 1.3 `Chart.yaml`

```yaml
apiVersion: v2
name: springboot-app
description: Generic Helm chart for Spring Boot microservices at BK
type: application
version: 0.1.0          # chart version — bump on breaking changes
appVersion: "latest"    # overridden per-deploy by CI
```

---

### 1.4 `values.yaml` — defaults

```yaml
# ── Image ──────────────────────────────────────────────
image:
  repository: ""          # required — set per project
  tag: "latest"
  pullPolicy: IfNotPresent
  pullSecrets: []

# ── Deployment ─────────────────────────────────────────
replicaCount: 1

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# ── App / Spring Boot ───────────────────────────────────
app:
  port: 8080
  profile: production
  jvmOpts: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

# ── Environment variables (extra, non-secret) ───────────
env: {}
  # SPRING_DATASOURCE_URL: "jdbc:postgresql://..."

# ── Secrets mounted as env vars ─────────────────────────
secretEnv: {}
  # secretName: my-db-secret
  # keys:
  #   DB_PASSWORD: password

# ── ConfigMap data ──────────────────────────────────────
configMap:
  enabled: false
  data: {}

# ── Service ─────────────────────────────────────────────
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# ── Ingress ─────────────────────────────────────────────
ingress:
  enabled: false
  className: "traefik"      # k3s default
  annotations: {}
  host: ""
  path: /
  pathType: Prefix
  tls:
    enabled: false
    secretName: ""

# ── Resources ───────────────────────────────────────────
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# ── Probes ──────────────────────────────────────────────
probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 20
    periodSeconds: 5
    failureThreshold: 3

# ── HPA ─────────────────────────────────────────────────
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70

# ── ServiceAccount ──────────────────────────────────────
serviceAccount:
  create: true
  name: ""
  annotations: {}

# ── Virtual Threads (Spring Boot 3.2+) ──────────────────
virtualThreads:
  enabled: true   # sets SPRING_THREADS_VIRTUAL_ENABLED=true
```

---

### 1.5 `templates/_helpers.tpl`

```gotmpl
{{/*
Expand the name of the chart.
*/}}
{{- define "springboot-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Full name: release-chart, capped at 63 chars.
*/}}
{{- define "springboot-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
Chart label
*/}}
{{- define "springboot-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "springboot-app.labels" -}}
helm.sh/chart: {{ include "springboot-app.chart" . }}
{{ include "springboot-app.selectorLabels" . }}
app.kubernetes.io/version: {{ .Values.image.tag | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "springboot-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "springboot-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
ServiceAccount name
*/}}
{{- define "springboot-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "springboot-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

### 1.6 `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "springboot-app.fullname" . }}
  labels:
    {{- include "springboot-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "springboot-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: {{ .Values.strategy.type }}
    {{- if eq .Values.strategy.type "RollingUpdate" }}
    rollingUpdate:
      maxSurge: {{ .Values.strategy.rollingUpdate.maxSurge }}
      maxUnavailable: {{ .Values.strategy.rollingUpdate.maxUnavailable }}
    {{- end }}
  template:
    metadata:
      labels:
        {{- include "springboot-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "springboot-app.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.app.port }}
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.app.profile | quote }}
            - name: JAVA_OPTS
              value: {{ .Values.app.jvmOpts | quote }}
            {{- if .Values.virtualThreads.enabled }}
            - name: SPRING_THREADS_VIRTUAL_ENABLED
              value: "true"
            {{- end }}
            {{- range $key, $val := .Values.env }}
            - name: {{ $key }}
              value: {{ $val | quote }}
            {{- end }}
            {{- if .Values.secretEnv }}
            {{- range .Values.secretEnv.keys }}
            - name: {{ .name }}
              valueFrom:
                secretKeyRef:
                  name: {{ $.Values.secretEnv.secretName }}
                  key: {{ .key }}
            {{- end }}
            {{- end }}
          {{- if .Values.configMap.enabled }}
          envFrom:
            - configMapRef:
                name: {{ include "springboot-app.fullname" . }}-config
          {{- end }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
            failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
            failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

### 1.7 `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "springboot-app.fullname" . }}
  labels:
    {{- include "springboot-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "springboot-app.selectorLabels" . | nindent 4 }}
```

---

### 1.8 `templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "springboot-app.fullname" . }}
  labels:
    {{- include "springboot-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls.enabled }}
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.tls.secretName }}
  {{- end }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: {{ .Values.ingress.path }}
            pathType: {{ .Values.ingress.pathType }}
            backend:
              service:
                name: {{ include "springboot-app.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

---

### 1.9 `templates/configmap.yaml`

```yaml
{{- if .Values.configMap.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "springboot-app.fullname" . }}-config
  labels:
    {{- include "springboot-app.labels" . | nindent 4 }}
data:
  {{- toYaml .Values.configMap.data | nindent 2 }}
{{- end }}
```

---

### 1.10 `templates/hpa.yaml`

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "springboot-app.fullname" . }}
  labels:
    {{- include "springboot-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "springboot-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
{{- end }}
```

---

## Part 2 — Publish the Chart Repo (GitLab)

### 2.1 Repo setup

```
gitlab.bk.rw/devops/helm-charts   (or GitHub, self-hosted Gitea, etc.)
```

Commit and push everything under `charts/springboot-app/`.

### 2.2 Package and push to GitLab Helm registry

GitLab has a built-in Helm chart registry per project. Use it.

```bash
# Package
helm package charts/springboot-app --destination dist/
# Output: dist/springboot-app-0.1.0.tgz

# Push to GitLab Helm registry
helm plugin install https://github.com/chartmuseum/helm-push
helm cm-push dist/springboot-app-0.1.0.tgz \
  oci://registry.gitlab.bk.rw/devops/helm-charts \
  --username $CI_REGISTRY_USER \
  --password $CI_REGISTRY_PASSWORD
```

> Alternatively, use GitLab's OCI-native registry (Helm 3.8+):

```bash
helm registry login registry.gitlab.bk.rw \
  -u $CI_REGISTRY_USER \
  -p $CI_REGISTRY_PASSWORD

helm push dist/springboot-app-0.1.0.tgz \
  oci://registry.gitlab.bk.rw/devops/helm-charts
```

### 2.3 CI pipeline for the chart repo itself

```yaml
# helm-charts/.gitlab-ci.yml
stages:
  - lint
  - package
  - publish

lint:
  stage: lint
  image: alpine/helm:3.14.0
  script:
    - helm lint charts/springboot-app

package-and-publish:
  stage: publish
  image: alpine/helm:3.14.0
  only:
    - main
  script:
    - helm package charts/springboot-app --destination dist/
    - helm registry login registry.gitlab.bk.rw
        -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
    - helm push dist/springboot-app-*.tgz
        oci://registry.gitlab.bk.rw/devops/helm-charts
```

---

## Part 3 — Update Your Service Project

### 3.1 New folder structure

```
my-service/
├── src/
│   └── main/kotlin/...
├── helm/
│   └── values.yaml          ← only project-specific overrides
├── Dockerfile
├── docker-compose.yml       ← local dev only
├── build.gradle.kts
└── .gitlab-ci.yml
```

> No full chart here — just `values.yaml`. The chart comes from the shared repo.

---

### 3.2 `helm/values.yaml` — project overrides

```yaml
# ── Override only what's different from chart defaults ──

image:
  repository: registry.gitlab.bk.rw/bk/my-service
  # tag is injected by CI: --set image.tag=$CI_COMMIT_SHORT_SHA

replicaCount: 2

app:
  profile: production
  port: 8080

env:
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-svc:5432/mydb"
  SPRING_DATASOURCE_HIKARI_MAXIMUM_POOL_SIZE: "10"

secretEnv:
  secretName: my-service-db-secret
  keys:
    - name: SPRING_DATASOURCE_USERNAME
      key: username
    - name: SPRING_DATASOURCE_PASSWORD
      key: password

ingress:
  enabled: true
  host: my-service.bk.rw
  tls:
    enabled: true
    secretName: my-service-tls

resources:
  requests:
    memory: "512Mi"
    cpu: "200m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

virtualThreads:
  enabled: true
```

---

### 3.3 `Dockerfile` — production image

```dockerfile
FROM eclipse-temurin:21-jre-alpine AS runtime

WORKDIR /app

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

ARG JAR_FILE=build/libs/*.jar
COPY ${JAR_FILE} app.jar

# Virtual threads + container-aware JVM
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

### 3.4 `build.gradle.kts` — Spring Boot Actuator (required for probes)

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    // ... rest of your deps
}
```

`application.yml` — expose health endpoints:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

---

## Part 4 — Reference the Shared Chart in CI

### 4.1 Add the Helm registry as a repo in CI

```bash
# In any job that does `helm upgrade`
helm registry login registry.gitlab.bk.rw \
  -u $CI_REGISTRY_USER \
  -p $CI_REGISTRY_PASSWORD

helm upgrade --install my-service \
  oci://registry.gitlab.bk.rw/devops/helm-charts/springboot-app \
  --version 0.1.0 \
  --values helm/values.yaml \
  --set image.tag=$CI_COMMIT_SHORT_SHA \
  --namespace my-namespace \
  --create-namespace \
  --wait --timeout 5m
```

> The `--version` pin keeps your service decoupled from chart updates. Bump deliberately.

---

## Part 5 — GitLab CI Pipeline for the Service

### 5.1 Full `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — my-service

stages:
  - build
  - test
  - security
  - docker
  - deploy

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"
  IMAGE_NAME: registry.gitlab.bk.rw/bk/my-service
  HELM_CHART_REPO: oci://registry.gitlab.bk.rw/devops/helm-charts
  HELM_CHART_NAME: springboot-app
  HELM_CHART_VERSION: "0.1.0"
  K8S_NAMESPACE: my-service
  KUBECONFIG: /tmp/k3s-kubeconfig

# ── Templates ─────────────────────────────────────────────────────────────────

.gradle-cache: &gradle-cache
  cache:
    key: "$CI_PROJECT_PATH-gradle"
    paths:
      - .gradle/
      - build/

.helm-setup: &helm-setup
  before_script:
    - helm registry login registry.gitlab.bk.rw
        -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
    - echo "$K3S_KUBECONFIG_B64" | base64 -d > $KUBECONFIG
    - export KUBECONFIG=$KUBECONFIG

# ── Build ─────────────────────────────────────────────────────────────────────

build:
  stage: build
  image: gradle:8.6-jdk21
  <<: *gradle-cache
  script:
    - gradle clean build -x test
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 hour

# ── Test ──────────────────────────────────────────────────────────────────────

test:
  stage: test
  image: gradle:8.6-jdk21
  <<: *gradle-cache
  services:
    - name: postgres:15-alpine
      alias: postgres
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
    SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres:5432/testdb"
    SPRING_DATASOURCE_USERNAME: test
    SPRING_DATASOURCE_PASSWORD: test
    SPRING_PROFILES_ACTIVE: test
  script:
    - gradle test
  artifacts:
    reports:
      junit: build/test-results/test/*.xml
    expire_in: 1 week

# ── Security ──────────────────────────────────────────────────────────────────

trivy-scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy fs --exit-code 0 --severity HIGH,CRITICAL
        --format template --template "@/contrib/gitlab.tpl"
        -o gl-dependency-scanning-report.json .
  artifacts:
    reports:
      dependency_scanning: gl-dependency-scanning-report.json
  allow_failure: true

# ── Docker ────────────────────────────────────────────────────────────────────

docker-build:
  stage: docker
  image: docker:24
  services:
    - docker:24-dind
  dependencies:
    - build
  before_script:
    - docker login registry.gitlab.bk.rw
        -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
  script:
    - |
      docker build \
        --build-arg JAR_FILE=build/libs/*.jar \
        -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA \
        -t $IMAGE_NAME:latest \
        .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_NAME:latest
  only:
    - main
    - develop

# ── Trivy image scan ──────────────────────────────────────────────────────────

docker-scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  dependencies:
    - docker-build
  script:
    - trivy image --exit-code 1 --severity CRITICAL
        $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
  allow_failure: true
  only:
    - main

# ── Deploy to staging ─────────────────────────────────────────────────────────

deploy-staging:
  stage: deploy
  image: alpine/helm:3.14.0
  <<: *helm-setup
  script:
    - |
      helm upgrade --install $K8S_NAMESPACE \
        $HELM_CHART_REPO/$HELM_CHART_NAME \
        --version $HELM_CHART_VERSION \
        --values helm/values.yaml \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set app.profile=staging \
        --set ingress.host=my-service-staging.bk.rw \
        --namespace ${K8S_NAMESPACE}-staging \
        --create-namespace \
        --wait --timeout 5m
  environment:
    name: staging
    url: https://my-service-staging.bk.rw
  only:
    - develop

# ── Deploy to production ──────────────────────────────────────────────────────

deploy-production:
  stage: deploy
  image: alpine/helm:3.14.0
  <<: *helm-setup
  script:
    - |
      helm upgrade --install $K8S_NAMESPACE \
        $HELM_CHART_REPO/$HELM_CHART_NAME \
        --version $HELM_CHART_VERSION \
        --values helm/values.yaml \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --namespace $K8S_NAMESPACE \
        --create-namespace \
        --wait --timeout 5m
  environment:
    name: production
    url: https://my-service.bk.rw
  when: manual          # ← requires a human click
  only:
    - main
```

---

## Part 6 — k3s Setup & GitLab Access

### 6.1 Get the kubeconfig from your k3s server

```bash
# On k3s server
sudo cat /etc/rancher/k3s/k3s.yaml
```

Edit the `server:` field to point to the public/internal IP (not `127.0.0.1`):

```yaml
server: https://YOUR_K3S_NODE_IP:6443
```

### 6.2 Encode and store in GitLab CI/CD variable

```bash
# On your macOS dev machine
cat k3s.yaml | base64 | pbcopy   # copies to clipboard
```

In GitLab → **Settings → CI/CD → Variables**:

| Key | Value | Protected | Masked |
|-----|-------|-----------|--------|
| `K3S_KUBECONFIG_B64` | `<base64 content>` | ✅ | ✅ |
| `CI_REGISTRY_USER` | GitLab username or token name | ✅ | ❌ |
| `CI_REGISTRY_PASSWORD` | GitLab token (registry scope) | ✅ | ✅ |

### 6.3 Create the namespace and image pull secret on k3s

```bash
# Run once per service/namespace
kubectl create namespace my-service

kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.bk.rw \
  --docker-username=<deploy-token-user> \
  --docker-password=<deploy-token-pass> \
  --namespace=my-service
```

Then reference it in `helm/values.yaml`:

```yaml
image:
  pullSecrets:
    - name: gitlab-registry
```

### 6.4 Create the DB secret on k3s (one-time, not in CI)

```bash
kubectl create secret generic my-service-db-secret \
  --from-literal=username=myuser \
  --from-literal=password=supersecret \
  --namespace=my-service
```

> Never put secret values in `values.yaml` or the repo. Create them directly on the cluster or via Vault.

---

## Part 7 — Local Dev Workflow (Docker Compose)

`docker-compose.yml` stays for local development only — it never touches k3s:

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/localdb
      SPRING_DATASOURCE_USERNAME: local
      SPRING_DATASOURCE_PASSWORD: local
      SPRING_THREADS_VIRTUAL_ENABLED: "true"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: localdb
      POSTGRES_USER: local
      POSTGRES_PASSWORD: local
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "local"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - pg_data:/var/lib/postgresql/data

volumes:
  pg_data:
```

---

## Part 8 — End-to-End Flow Summary

```
Developer pushes to `develop`
    │
    ▼
GitLab CI
  [build]     gradle clean build
  [test]      gradle test (Postgres service container)
  [security]  trivy fs scan
  [docker]    build + push image → GitLab Registry
  [docker-scan] trivy image scan
  [deploy]    helm upgrade --install
                ← pulls chart from helm-charts OCI registry
                ← uses helm/values.yaml from this repo
                ← --set image.tag=$CI_COMMIT_SHORT_SHA
                → deploys to k3s staging namespace
    │
    ▼  (merge to main → manual trigger)
  [deploy]    helm upgrade --install → k3s production namespace
```

---

## Quick Cheatsheet

```bash
# Add new project: 3 files needed
#   Dockerfile
#   helm/values.yaml          (override image, host, env, secrets)
#   .gitlab-ci.yml            (copy template, change variables block)

# Upgrade chart version across all services
# → bump HELM_CHART_VERSION in each .gitlab-ci.yml
# → or use a GitLab group-level variable HELM_CHART_VERSION

# Rollback
helm rollback my-service 0 --namespace my-service   # 0 = previous release

# Dry-run before deploy
helm upgrade --install my-service \
  oci://registry.gitlab.bk.rw/devops/helm-charts/springboot-app \
  --version 0.1.0 \
  --values helm/values.yaml \
  --set image.tag=abc123 \
  --namespace my-service \
  --dry-run --debug
```
