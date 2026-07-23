---
layout: default
title: "Lab 07.1: Packaging Distribution"
nav_order: 11
parent: "Модуль 7: Подготовка к продакшену"
grand_parent: Модули
mermaid: true
---

# Лабораторная 7.1: Упаковка вашего оператора

**Связанный урок:** [Урок 7.1: Упаковка и распространение](../lessons/01-packaging-distribution.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: RBAC →](lab-02-rbac-security.md)

## Цели

- Собрать образ контейнера для оператора
- Создать Helm-чарт для развёртывания
- Правильно расставить теги и версии образов
- Опубликовать в реестр контейнеров

## Предварительные требования

- Завершение [Модуля 6](../../module-06/README.md)
- Готовый оператор Database
- Установленный Docker или Podman
- Доступ к реестру контейнеров (или используйте kind локально)

## Упражнение 1: сборка образа контейнера

Kubebuilder уже сгенерировал готовый к продакшену Dockerfile при создании каркаса вашего проекта. Изучим и используем его.

### Задача 1.1: изучите Dockerfile, сгенерированный Kubebuilder

Kubebuilder создаёт `Dockerfile` в корне проекта. Изучите его:

```bash
# Navigate to your operator project from module 3
cd ~/postgres-operator

# View the Dockerfile
cat Dockerfile
```

Сгенерированный Dockerfile должен выглядеть так:

```dockerfile
# Build stage
FROM golang:1.24 as builder
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace
COPY go.mod go.mod
COPY go.sum go.sum
RUN go mod download

# Copy the go source
COPY cmd/main.go cmd/main.go
COPY api/ api/
COPY internal/ internal/

# Build
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH:-amd64} go build -a -o manager cmd/main.go

# Runtime stage
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532

ENTRYPOINT ["/manager"]
```

**Важно:** копируется весь каталог `internal/`, который включает:
- `internal/controller/` — логику согласования вашего контроллера
- `internal/webhook/` — обработчики вебхуков (созданные в Модуле 5)

### Задача 1.2: соберите образ с помощью Makefile

Kubebuilder предоставляет цели Makefile для сборки образов:

```bash
# Build the image using kubebuilder's make target
make docker-build IMG=postgres-operator:v0.1.0

# For kind, load image into the cluster
kind load docker-image postgres-operator:v0.1.0 --name k8s-operators-course

# Verify image is available in kind
docker exec -it k8s-operators-course-control-plane crictl images | grep postgres-operator
```

## Упражнение 2: развёртывание с помощью Kustomize от Kubebuilder (рекомендуется)

Kubebuilder по умолчанию использует Kustomize для развёртывания. Это рекомендуемый подход.

### Задача 2.1: изучите конфигурацию Kustomize

```bash
# Explore the config directory structure
ls -la config/

# Key directories:
# config/crd/       - CRD definitions
# config/default/   - Main kustomization
# config/manager/   - Controller deployment
# config/rbac/      - RBAC rules
```

### Задача 2.2: разверните с помощью Kustomize

```bash
# Install CRDs
make install

# Deploy the operator (builds and deploys)
make deploy IMG=postgres-operator:v0.1.0

# Verify deployment
kubectl get deployment -n postgres-operator-system
kubectl get pods -n postgres-operator-system
```

### Задача 2.3: просмотрите сгенерированные манифесты

```bash
# Preview what will be deployed
kustomize build config/default

# Or using make target
make build-installer IMG=postgres-operator:v0.1.0
```

## Упражнение 3: создание Helm-чарта из Kustomize

Для более широкого распространения вы можете сгенерировать Helm-чарт из ваших манифестов Kustomize. Чарт должен включать **все компоненты оператора**:

- **CRD** — определения пользовательских ресурсов (из `config/crd/`)
- **RBAC** — ServiceAccount, ClusterRole, ClusterRoleBinding (из `config/rbac/`)
- **Deployment** — менеджер-контроллер (из `config/manager/`)
- **Вебхуки** — если созданы в Модуле 5 (из `config/webhook/`)

**Важно:** Helm-чарт только с Deployment работать не будет! Оператору нужны все эти компоненты для работы.

### Задача 3.1: добавьте цель Makefile для Helm-чарта

Добавьте эти цели в ваш `Makefile`:

```makefile
# Helm chart configuration
CHART_NAME ?= postgres-operator
CHART_VERSION ?= 0.1.0
CHART_DIR ?= charts/$(CHART_NAME)

##@ Helm

.PHONY: helm-chart
helm-chart: manifests kustomize ## Generate Helm chart from Kustomize (includes CRDs, RBAC, Deployment, Webhooks)
	@echo "Generating Helm chart with ALL operator components..."
	@mkdir -p $(CHART_DIR)/templates
	@# Create Chart.yaml
	@printf '%s\n' \
		'apiVersion: v2' \
		'name: $(CHART_NAME)' \
		'description: A Helm chart for $(CHART_NAME) - includes CRDs, RBAC, and webhooks' \
		'type: application' \
		'version: $(CHART_VERSION)' \
		'appVersion: "$(VERSION)"' \
		> $(CHART_DIR)/Chart.yaml
	@# Create values.yaml
	@printf '%s\n' \
		'image:' \
		'  repository: $(IMAGE_TAG_BASE)' \
		'  tag: $(VERSION)' \
		'  pullPolicy: IfNotPresent' \
		'' \
		'replicaCount: 1' \
		'' \
		'resources:' \
		'  limits:' \
		'    cpu: 500m' \
		'    memory: 128Mi' \
		'  requests:' \
		'    cpu: 10m' \
		'    memory: 64Mi' \
		'' \
		'leaderElection:' \
		'  enabled: false' \
		'' \
		'namespace: $(CHART_NAME)-system' \
		> $(CHART_DIR)/values.yaml
	@# Generate ALL manifests from kustomize (CRDs, RBAC, Deployment, Webhooks)
	@cd config/manager && $(KUSTOMIZE) edit set image controller=$(IMG)
	@$(KUSTOMIZE) build config/default > $(CHART_DIR)/templates/manifests.yaml
	@echo "Helm chart generated at $(CHART_DIR)"
	@echo "Contents include: CRDs, RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding), Deployment, Webhooks"

.PHONY: helm-package
helm-package: helm-chart ## Package Helm chart
	@mkdir -p dist
	helm package $(CHART_DIR) -d dist/

.PHONY: helm-lint
helm-lint: helm-chart ## Lint Helm chart
	helm lint $(CHART_DIR)

.PHONY: helm-template
helm-template: helm-chart ## Render Helm templates locally
	helm template $(CHART_NAME) $(CHART_DIR)

.PHONY: helm-install
helm-install: helm-chart ## Install Helm chart to cluster
	helm upgrade --install $(CHART_NAME) $(CHART_DIR) \
		--namespace $(CHART_NAME)-system \
		--create-namespace

.PHONY: helm-uninstall
helm-uninstall: ## Uninstall Helm chart
	helm uninstall $(CHART_NAME) --namespace $(CHART_NAME)-system
```

### Задача 3.2: сгенерируйте и проверьте Helm-чарт

```bash
# Generate Helm chart from Kustomize
make helm-chart IMG=postgres-operator:v0.1.0

# Verify the chart structure
ls -la charts/postgres-operator/
ls -la charts/postgres-operator/templates/

# IMPORTANT: Verify the generated manifests include all components
echo "=== Checking for CRDs ==="
grep -c "kind: CustomResourceDefinition" charts/postgres-operator/templates/manifests.yaml

echo "=== Checking for RBAC ==="
grep -c "kind: ServiceAccount" charts/postgres-operator/templates/manifests.yaml
grep -c "kind: ClusterRole" charts/postgres-operator/templates/manifests.yaml
grep -c "kind: ClusterRoleBinding" charts/postgres-operator/templates/manifests.yaml

echo "=== Checking for Deployment ==="
grep -c "kind: Deployment" charts/postgres-operator/templates/manifests.yaml

# Lint the chart
make helm-lint

# Preview ALL rendered templates
make helm-template | head -100
```

### Задача 3.3: упакуйте и протестируйте чарт

```bash
# Package the chart
make helm-package

# List packaged charts
ls -la dist/

# Test install (optional)
make helm-install

# Verify deployment
kubectl get pods -n postgres-operator-system

# Cleanup
make helm-uninstall
```

## Упражнение 4: GitHub Actions для CI/CD

Автоматизируйте публикацию чарта с помощью GitHub Actions.

### Задача 4.1: создайте workflow GitHub Actions

Создайте `.github/workflows/release.yaml`:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.24'

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build and push Docker image
        run: |
          make docker-build docker-push IMG=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.VERSION }}

  helm-release:
    runs-on: ubuntu-latest
    needs: build-and-push
    permissions:
      contents: write
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.12.0

      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Generate Helm chart
        run: |
          make helm-chart \
            IMG=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }} \
            CHART_VERSION=${{ steps.version.outputs.VERSION }} \
            VERSION=${{ steps.version.outputs.VERSION }}

      - name: Package Helm chart
        run: make helm-package

      - name: Push Helm chart to GHCR
        run: |
          helm push dist/*.tgz oci://${{ env.REGISTRY }}/${{ github.repository_owner }}/charts

      - name: Upload chart as release artifact
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*.tgz
```

### Задача 4.2: создайте репозиторий Helm-чартов (альтернатива)

Для традиционного репозитория Helm на GitHub Pages создайте `.github/workflows/helm-release.yaml`:

```yaml
name: Helm Chart Release

on:
  push:
    branches:
      - main
    paths:
      - 'charts/**'
      - '.github/workflows/helm-release.yaml'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Install Helm
        uses: azure/setup-helm@v3

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.6.0
        with:
          charts_dir: charts
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

### Задача 4.3: добавьте документацию репозитория

Создайте `charts/README.md`:

```markdown
# Database Operator Helm Chart

## Installation

### Using OCI Registry (GHCR)

```bash
helm install postgres-operator oci://ghcr.io/YOUR_USERNAME/charts/postgres-operator --version 0.1.0
```

### Using Helm Repository

```bash
# Add the repository
helm repo add postgres-operator https://YOUR_USERNAME.github.io/postgres-operator

# Update repositories
helm repo update

# Install
helm install postgres-operator postgres-operator/postgres-operator
```

## Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Image repository | `ghcr.io/YOUR_USERNAME/postgres-operator` |
| `image.tag` | Image tag | `v0.1.0` |
| `replicaCount` | Number of replicas | `1` |
| `leaderElection.enabled` | Enable leader election | `false` |
| `resources.limits.cpu` | CPU limit | `500m` |
| `resources.limits.memory` | Memory limit | `128Mi` |


### Задача 4.4: протестируйте workflow локально (опционально)

```bash
# Create a test tag
git tag v0.1.0
git push origin v0.1.0

# Watch the Actions tab in GitHub for workflow execution
```

## Упражнение 5: версионирование и теги

### Задача 5.1: задайте версию вашего оператора

Обновите версию в `Makefile`:

```makefile
# Image URL to use all building/pushing image targets
IMG ?= postgres-operator:v0.1.0
VERSION ?= 0.1.0
```

Или укажите её при сборке:

```bash
# Build with specific version
make docker-build IMG=postgres-operator:v0.1.0

# Build for multiple architectures (if needed)
make docker-buildx IMG=postgres-operator:v0.1.0
```

### Задача 5.2: опубликуйте в реестр

```bash
# Tag for your registry
docker tag postgres-operator:v0.1.0 ghcr.io/your-username/postgres-operator:v0.1.0

# Push (requires authentication)
docker push ghcr.io/your-username/postgres-operator:v0.1.0

# Or use make target
make docker-push IMG=ghcr.io/your-username/postgres-operator:v0.1.0
```

### Задача 5.3: разверните конкретную версию

```bash
# Deploy with Kustomize
make deploy IMG=ghcr.io/your-username/postgres-operator:v0.1.0

# Or deploy with Helm
make helm-install IMG=ghcr.io/your-username/postgres-operator:v0.1.0

# Verify
kubectl get deployment -n postgres-operator-system -o yaml | grep image:
```

## Очистка

```bash
# Undeploy the operator (Kustomize)
make undeploy

# Or undeploy with Helm
make helm-uninstall

# Uninstall CRDs
make uninstall

# Remove local images (optional)
docker rmi postgres-operator:v0.1.0

# Clean up generated charts
rm -rf charts/ dist/
```

## Итоги лабораторной

В этой лабораторной вы:
- Изучили Dockerfile, сгенерированный kubebuilder
- Собрали образы контейнеров с помощью `make docker-build`
- Развернули с использованием конфигурации Kustomize от kubebuilder
- Создали цель make для генерации Helm-чартов из Kustomize
- Настроили GitHub Actions для автоматизированных релизов
- Правильно расставили теги и версии образов

## Ключевые уроки

1. Kubebuilder генерирует готовый к продакшену Dockerfile
2. Используйте `make docker-build` и `make docker-push` для образов
3. Используйте `make deploy` для развёртывания на основе Kustomize
4. `make helm-chart` генерирует Helm-чарты из манифестов Kustomize
5. GitHub Actions автоматизируют публикацию образов и чартов
6. OCI-реестры (например, GHCR) могут хранить и образы, И Helm-чарты
7. Семантическое версионирование отслеживает релизы оператора

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Dockerfile](../solutions/Dockerfile) — готовый к продакшену многоэтапный Dockerfile
- [Helm Chart](../solutions/helm-chart/) — полный Helm-чарт (Chart.yaml, values.yaml, templates)
- [GitHub Actions](../solutions/github-actions/) — CI/CD-workflows для релизов

## Дальнейшие шаги

Теперь давайте настроим корректный RBAC и безопасность!

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-packaging-distribution.md) | [Следующая лабораторная: RBAC →](lab-02-rbac-security.md)
