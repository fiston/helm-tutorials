# Spring Boot → k3s Deployment via Shared Helm Chart Templates

> Stack: Kotlin · Spring Boot · Docker · k3s · Helm · gitlab.com CI · GitVersion  
> Pattern: one shared `helm-charts` repo consumed by N microservice repos  
> Versioning: GitVersion drives every image tag, Helm chart version, and `appVersion` automatically from git history  
> Registry: all built-in gitlab.com variables (`$CI_REGISTRY`, `$CI_REGISTRY_IMAGE`, `$CI_REGISTRY_USER`, `$CI_REGISTRY_PASSWORD`) — nothing to create manually

---

## Overview

```bash
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

## gitlab.com Built-in Registry Variables

gitlab.com automatically injects these into **every pipeline job** — you never define them:

| Variable                | Resolves to                                 |
|-------------------------|---------------------------------------------|
| `$CI_REGISTRY`          | `registry.gitlab.com`                       |
| `$CI_REGISTRY_IMAGE`    | `registry.gitlab.com/<namespace>/<project>` |
| `$CI_REGISTRY_USER`     | `gitlab-ci-token` (ephemeral per job)       |
| `$CI_REGISTRY_PASSWORD` | Job token, rotated every pipeline           |

Use these everywhere instead of hardcoding registry URLs. The only prerequisite: **Settings → CI/CD → Token Access → Job token scope** must allow the project to push to its own registry (enabled by default on gitlab.com).

---

## Part 0 — GitVersion Setup (Both Repos)

GitVersion reads your git tags and branch names to produce a deterministic semantic version (`1.3.0`, `1.3.0-alpha.4`, `1.3.0-rc.1`, etc.) that flows into Docker image tags, Helm `appVersion`, and chart packaging — zero manual version bumping.

### 0.1 How it works in this setup

```bash
git tag v1.2.0  →  main branch   →  GITVERSION_SEMVER = 1.2.1   (patch auto-increment)
                   develop branch →  GITVERSION_SEMVER = 1.3.0-alpha.3
                   feature/xyz    →  GITVERSION_SEMVER = 1.3.0-feature-xyz.1
```

Every CI job gets the version as environment variables via a `dotenv` artifact:

```bash
GITVERSION_SEMVER=1.3.0-alpha.3
GITVERSION_MAJORMINORPATCH=1.3.0
GITVERSION_FULLSEMVER=1.3.0-alpha.3
GITVERSION_INFORMATIOALVERSION=1.3.0-alpha.3+Branch.develop.Sha.abc1234
```

### 0.2 `GitVersion.yml` — place this in the **root of both repos**

```yaml
# GitVersion.yml
assembly-versioning-scheme: MajorMinorPatch
assembly-file-versioning-scheme: MajorMinorPatch
mode: Mainline
tag-prefix: "v"

branches:
  main:
    regex: ^main$
    label: ""                    # no pre-release label → clean 1.2.3
    increment: Patch
    prevent-increment-of-merged-branch-version: true
    track-merge-target: false

  develop:
    regex: ^develop$
    label: "alpha"               # → 1.3.0-alpha.{commits}
    increment: Minor
    track-merge-target: true

  release:
    regex: ^release[/-]
    label: "rc"                  # → 1.3.0-rc.{commits}
    increment: Minor

  feature:
    regex: ^feature[/-]
    label: "feature-{BranchName}"
    increment: Minor

  hotfix:
    regex: ^hotfix[/-]
    label: "hotfix"
    increment: Patch

ignore:
  sha: []
```

### 0.3 Install GitVersion (macOS dev machine)

```bash
# Homebrew
brew install gitversion

# Or via .NET tool (if you already have dotnet)
dotnet tool install --global GitVersion.Tool

# Verify
gitversion /version
```

### 0.4 Run locally to preview version

```bash
# Must be run inside a git repo with full history
git fetch --tags
gitversion /output json
```

> **Critical:** GitLab CI must clone with full git history — set `GIT_DEPTH: 0` in any job that runs GitVersion. Without this, GitVersion cannot walk the tag history and will fall back to `0.1.0`.

---

## Part 1 — Create the Shared Helm Chart Template Repo

### 1.1 Scaffold the chart

```bash
helm create springboot-app
```

This generates boilerplate. Clean it up and replace with the structure below.

### 1.2 Final folder structure

```bash
helm-charts/
├── README.md
├── GitVersion.yml                 ← chart repo versioning
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
version: 0.0.0          # placeholder — overridden by GitVersion in CI via --set version
appVersion: "0.0.0"     # placeholder — overridden per-deploy via --set appVersion
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

```bash
gitlab.com/your-group/helm-charts
```

Commit and push everything under `charts/springboot-app/`.

### 2.2 Package and push to GitLab Helm registry

GitLab has a built-in Helm chart registry per project. Use it.

```bash
# Package
helm package charts/springboot-app --destination dist/
# Output: dist/springboot-app-0.1.0.tgz

# Push to GitLab Helm registry (OCI-native, Helm 3.8+)
helm registry login $CI_REGISTRY \
  -u $CI_REGISTRY_USER \
  -p $CI_REGISTRY_PASSWORD

helm push dist/springboot-app-0.1.0.tgz \
  oci://$CI_REGISTRY/$CI_PROJECT_PATH
```

> `$CI_PROJECT_PATH` resolves to `your-group/helm-charts` — no hardcoding needed.
> Alternatively, use the `helm-push` plugin with ChartMuseum if you prefer HTTP-based repos:

```bash
helm plugin install https://github.com/chartmuseum/helm-push
helm cm-push dist/springboot-app-0.1.0.tgz \
  oci://$CI_REGISTRY/$CI_PROJECT_PATH \
  --username $CI_REGISTRY_USER \
  --password $CI_REGISTRY_PASSWORD
```

### 2.3 CI pipeline for the chart repo itself

```yaml
# helm-charts/.gitlab-ci.yml
stages:
  - version
  - lint
  - package
  - publish

variables:
  GIT_DEPTH: 0                    # required — GitVersion needs full history
  GIT_FETCH_EXTRA_FLAGS: --tags   # required — GitVersion needs all tags
  # $CI_REGISTRY, $CI_REGISTRY_USER, $CI_REGISTRY_PASSWORD are auto-injected by gitlab.com

# ── Reusable template ─────────────────────────────────────────────────────────

.gitversion-vars: &gitversion-vars
  image: gittools/gitversion:6.0.0-alpine.3.18-7.0
  before_script:
    - gitversion /output buildserver /nofetch > gitversion.env
    - source gitversion.env || export $(cat gitversion.env | xargs)
    - echo "Chart version → $GITVERSION_SEMVER"

# ── Version ───────────────────────────────────────────────────────────────────

gitversion:
  stage: version
  image: gittools/gitversion:6.0.0-alpine.3.18-7.0
  script:
    - gitversion /output buildserver /nofetch | tee gitversion.env
  artifacts:
    reports:
      dotenv: gitversion.env    # exports GITVERSION_* to all downstream jobs
    expire_in: 1 hour

# ── Lint ──────────────────────────────────────────────────────────────────────

lint:
  stage: lint
  image: alpine/helm:3.14.0
  needs: [gitversion]
  script:
    - echo "Linting chart at version $GITVERSION_SEMVER"
    - helm lint charts/springboot-app

# ── Package & Publish ─────────────────────────────────────────────────────────

package-and-publish:
  stage: publish
  image: alpine/helm:3.14.0
  needs: [gitversion, lint]
  only:
    - main
    - develop
    - /^release\/.*/
  script:
    # Inject GitVersion into Chart.yaml before packaging
    - |
      sed -i "s/^version:.*/version: ${GITVERSION_SEMVER}/" charts/springboot-app/Chart.yaml
      sed -i "s/^appVersion:.*/appVersion: \"${GITVERSION_SEMVER}\"/" charts/springboot-app/Chart.yaml
    - helm package charts/springboot-app --destination dist/
    - helm registry login $CI_REGISTRY
        -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
    - helm push dist/springboot-app-${GITVERSION_SEMVER}.tgz
        oci://$CI_REGISTRY/$CI_PROJECT_PATH
    - echo "Published springboot-app:${GITVERSION_SEMVER} → oci://$CI_REGISTRY/$CI_PROJECT_PATH"
```

---

## Part 3 — Update Your Service Project

### 3.1 New folder structure

```bash
my-service/
├── src/
│   └── main/kotlin/...
├── helm/
│   └── values.yaml          ← only project-specific overrides
├── GitVersion.yml           ← same file as in helm-charts repo
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
  repository: $CI_REGISTRY_IMAGE   # auto-resolves to registry.gitlab.com/<namespace>/<project>
  # tag is injected by CI: --set image.tag=$GITVERSION_SEMVER
  # e.g. 1.3.0 on main, 1.3.0-alpha.4 on develop

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
ARG APP_VERSION=0.0.0           # injected by CI: --build-arg APP_VERSION=$GITVERSION_SEMVER
COPY ${JAR_FILE} app.jar

# Embed version as OCI label (visible via `docker inspect`)
LABEL org.opencontainers.image.version="${APP_VERSION}"
LABEL org.opencontainers.image.source="https://gitlab.com/${CI_PROJECT_PATH}"

# Virtual threads + container-aware JVM
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -Djava.security.egd=file:/dev/./urandom"

# Expose version to Spring Boot's /actuator/info
ENV APP_VERSION=${APP_VERSION}

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

`application.yml` — expose health endpoints and version:

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
  info:
    env:
      enabled: true   # exposes info.* env vars at /actuator/info

info:
  app:
    version: ${APP_VERSION:local}   # set via Dockerfile ENV APP_VERSION
    name: my-service
```

---

## Part 4 — Reference the Shared Chart in CI

### 4.1 Add the Helm registry as a repo in CI

```bash
# In any job that does `helm upgrade`
# All $CI_REGISTRY_* and $GITVERSION_SEMVER are auto-injected — no manual setup needed

helm registry login $CI_REGISTRY \
  -u $CI_REGISTRY_USER \
  -p $CI_REGISTRY_PASSWORD

helm upgrade --install my-service \
  oci://$CI_REGISTRY/your-group/helm-charts/springboot-app \
  --version $HELM_CHART_VERSION \            # chart version — pinned in variables
  --values helm/values.yaml \
  --set image.tag=$GITVERSION_SEMVER \       # ← from GitVersion dotenv artifact
  --set image.repository=$CI_REGISTRY_IMAGE \# ← built-in, no hardcoding
  --set appVersion=$GITVERSION_SEMVER \      # ← visible in helm history
  --namespace my-namespace \
  --create-namespace \
  --wait --timeout 5m
```

> Pin `HELM_CHART_VERSION` to a specific chart release (e.g. `1.2.0`). The service's own `GITVERSION_SEMVER` is independent of the chart version — they version different things.

---

## Part 5 — GitLab CI Pipeline for the Service

### 5.1 Full `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — my-service

stages:
  - version
  - build
  - test
  - security
  - docker
  - deploy

variables:
  GIT_DEPTH: 0                    # required — GitVersion needs full git history
  GIT_FETCH_EXTRA_FLAGS: --tags   # required — GitVersion needs all tags
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"
  # $CI_REGISTRY          → registry.gitlab.com              (auto-injected)
  # $CI_REGISTRY_IMAGE    → registry.gitlab.com/<ns>/<proj>  (auto-injected)
  # $CI_REGISTRY_USER     → gitlab-ci-token                  (auto-injected)
  # $CI_REGISTRY_PASSWORD → ephemeral job token              (auto-injected)
  HELM_CHART_REPO: oci://registry.gitlab.com/your-group/helm-charts
  HELM_CHART_NAME: springboot-app
  HELM_CHART_VERSION: "1.0.0"     # pin to a published chart version
  K8S_NAMESPACE: my-service
  KUBECONFIG: /tmp/k3s-kubeconfig

# ── Reusable templates ────────────────────────────────────────────────────────

.gradle-cache: &gradle-cache
  cache:
    key: "$CI_PROJECT_PATH-gradle"
    paths:
      - .gradle/
      - build/

.helm-setup: &helm-setup
  before_script:
    - helm registry login $CI_REGISTRY
        -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
    - echo "$K3S_KUBECONFIG_B64" | base64 -d > $KUBECONFIG
    - export KUBECONFIG=$KUBECONFIG
    - echo "Deploying image tag → $GITVERSION_SEMVER"

# ── Version ───────────────────────────────────────────────────────────────────

gitversion:
  stage: version
  image: gittools/gitversion:6.0.0-alpine.3.18-7.0
  script:
    # Output all GITVERSION_* vars into a dotenv file for downstream jobs
    - gitversion /output buildserver /nofetch | tee gitversion.env
    - echo "Resolved version → $(grep GITVERSION_SEMVER gitversion.env)"
  artifacts:
    reports:
      dotenv: gitversion.env      # all downstream jobs get GITVERSION_* in env
    expire_in: 1 hour

# ── Build ─────────────────────────────────────────────────────────────────────

build:
  stage: build
  image: gradle:8.6-jdk21
  needs: [gitversion]
  <<: *gradle-cache
  script:
    # Pass version to Gradle so it's baked into the JAR manifest
    - gradle clean build -x test -Pversion=$GITVERSION_SEMVER
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 hour

# ── Test ──────────────────────────────────────────────────────────────────────

test:
  stage: test
  image: gradle:8.6-jdk21
  needs: [gitversion, build]
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
  needs: [gitversion]
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
  needs: [gitversion, build, test]
  services:
    - docker:24-dind
  before_script:
    - docker login $CI_REGISTRY
        -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
  script:
    - |
      docker build \
        --build-arg JAR_FILE=build/libs/*.jar \
        --build-arg APP_VERSION=$GITVERSION_SEMVER \
        -t $CI_REGISTRY_IMAGE:$GITVERSION_SEMVER \
        -t $CI_REGISTRY_IMAGE:latest \
        .
    - docker push $CI_REGISTRY_IMAGE:$GITVERSION_SEMVER
    - docker push $CI_REGISTRY_IMAGE:latest
    - echo "Pushed $CI_REGISTRY_IMAGE:$GITVERSION_SEMVER"
  only:
    - main
    - develop

# ── Trivy image scan ──────────────────────────────────────────────────────────

docker-scan:
  stage: security
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  needs: [gitversion, docker-build]
  script:
    - trivy image --exit-code 1 --severity CRITICAL
        $CI_REGISTRY_IMAGE:$GITVERSION_SEMVER
  allow_failure: true
  only:
    - main

# ── Deploy to staging ─────────────────────────────────────────────────────────

deploy-staging:
  stage: deploy
  image: alpine/helm:3.14.0
  needs: [gitversion, docker-build]
  <<: *helm-setup
  script:
    - |
      helm upgrade --install $K8S_NAMESPACE \
        $HELM_CHART_REPO/$HELM_CHART_NAME \
        --version $HELM_CHART_VERSION \
        --values helm/values.yaml \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$GITVERSION_SEMVER \
        --set appVersion=$GITVERSION_SEMVER \
        --set app.profile=staging \
        --set ingress.host=my-service-staging.example.com \
        --namespace ${K8S_NAMESPACE}-staging \
        --create-namespace \
        --wait --timeout 5m
  environment:
    name: staging
    url: https://my-service-staging.example.com
  only:
    - develop

# ── Deploy to production ──────────────────────────────────────────────────────

deploy-production:
  stage: deploy
  image: alpine/helm:3.14.0
  needs: [gitversion, docker-build]
  <<: *helm-setup
  script:
    - |
      helm upgrade --install $K8S_NAMESPACE \
        $HELM_CHART_REPO/$HELM_CHART_NAME \
        --version $HELM_CHART_VERSION \
        --values helm/values.yaml \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$GITVERSION_SEMVER \
        --set appVersion=$GITVERSION_SEMVER \
        --namespace $K8S_NAMESPACE \
        --create-namespace \
        --wait --timeout 5m
  environment:
    name: production
    url: https://my-service.example.com
  when: manual          # ← requires a human click in GitLab UI
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

| Key                   | Value              | Protected | Masked | Notes                                              |
|-----------------------|--------------------|-----------|--------|----------------------------------------------------|
| `K3S_KUBECONFIG_B64`  | `<base64 content>` | ✅        | ✅     | Must create — your k3s kubeconfig                  |
| `HELM_CHART_VERSION`  | e.g. `1.0.0`       | ❌        | ❌     | Set at **group level** to share across all services|

> `$CI_REGISTRY`, `$CI_REGISTRY_USER`, `$CI_REGISTRY_PASSWORD`, and `$CI_REGISTRY_IMAGE` are **predefined by gitlab.com** — do not create them. They are automatically available in every job.
> `HELM_CHART_VERSION` set at the **GitLab group level** means all service repos inherit it automatically. Bump it there once when you publish a new chart version.

### 6.3 Create the namespace and image pull secret on k3s

```bash
# Run once per service/namespace
kubectl create namespace my-service

# Use a GitLab Deploy Token (not your personal token) for the pull secret
# Create one at: gitlab.com/your-group/helm-charts → Settings → Repository → Deploy tokens
# Scope: read_registry
kubectl create secret docker-registry gitlab-registry \
  --docker-server=registry.gitlab.com \
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

```bash
Developer pushes to `develop`
    │
    ▼
GitLab CI
  [version]   gitversion → resolves 1.3.0-alpha.4
                → exports GITVERSION_SEMVER via dotenv artifact
  [build]     gradle clean build -Pversion=1.3.0-alpha.4
  [test]      gradle test (Postgres service container)
  [security]  trivy fs scan
  [docker]    docker build --build-arg APP_VERSION=1.3.0-alpha.4
              → image tagged :1.3.0-alpha.4 + :latest
              → pushed to GitLab Registry
  [docker-scan] trivy image scan on :1.3.0-alpha.4
  [deploy]    helm upgrade --install
                ← chart: oci://…/springboot-app:1.0.0  (HELM_CHART_VERSION pin)
                ← --set image.tag=1.3.0-alpha.4
                ← --set appVersion=1.3.0-alpha.4
                → deploys to k3s staging namespace
    │
    ▼  (merge to main → manual trigger)
  [version]   gitversion → resolves 1.3.0  (clean semver, no pre-release label)
  [build/test/docker] ... same flow, tagged :1.3.0
  [deploy]    helm upgrade --install → k3s production namespace
```

### GitVersion tag workflow for releases

```bash
# On main after merge — tag to drive the next version
git tag v1.3.0
git push origin v1.3.0
# → next build on main resolves GITVERSION_SEMVER = 1.3.0
# → next commit after tag resolves to 1.3.1 (patch increment)
```

---

## Quick Cheatsheet

```bash
# Add new project: 4 files needed
#   GitVersion.yml            (copy from this guide — same for all services)
#   Dockerfile
#   helm/values.yaml          (override image, host, env, secrets)
#   .gitlab-ci.yml            (copy template, change variables block)

# Preview GitVersion locally
git fetch --tags
gitversion /output json | jq '{semver: .SemVer, full: .FullSemVer, info: .InformationalVersion}'

# Bump chart version for all services at once
# → update HELM_CHART_VERSION in GitLab group-level CI/CD variables

# Rollback
helm rollback my-service 0 --namespace my-service   # 0 = previous release

# Check what version is running
helm history my-service --namespace my-service
kubectl get deployment my-service -n my-service \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Dry-run before deploy
# Replace YOUR_GROUP and use a real GITVERSION_SEMVER value
helm registry login registry.gitlab.com \
  -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD

helm upgrade --install my-service \
  oci://registry.gitlab.com/your-group/helm-charts/springboot-app \
  --version 1.0.0 \
  --values helm/values.yaml \
  --set image.repository=registry.gitlab.com/your-group/my-service \
  --set image.tag=1.3.0-alpha.4 \
  --set appVersion=1.3.0-alpha.4 \
  --namespace my-service \
  --dry-run --debug

# Create a release tag to drive clean semver on main
git tag v1.3.0 && git push origin v1.3.0
```
