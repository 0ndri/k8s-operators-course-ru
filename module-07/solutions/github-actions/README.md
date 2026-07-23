# Решения GitHub Actions для Модуля 7

Этот каталог содержит workflows GitHub Actions для автоматизации релизов оператора.

## Workflows

### `release.yaml` — основной workflow релиза

Запускается по тегам версий (например, `v0.1.0`) и:

1. **Собирает и публикует Docker-образ** в GitHub Container Registry (GHCR)
   - Поддержка нескольких архитектур (amd64, arm64)
   - Теги семантических версий
   - Теги на основе SHA для трассируемости

2. **Генерирует и публикует Helm-чарт** в OCI-реестр GHCR
   - Использует цель `make helm-chart`
   - Публикует в `oci://ghcr.io/<owner>/charts`
   - Прикрепляет чарт как артефакт релиза

3. **Создаёт манифест-установщик**
   - Установка одним файлом через `kubectl apply -f`
   - Прикрепляется к релизу GitHub

### `helm-gh-pages.yaml` — репозиторий Helm на GitHub Pages

Альтернативный подход с использованием GitHub Pages:

- Запускается при изменениях в каталоге `charts/`
- Использует helm/chart-releaser-action
- Публикует в репозиторий Helm на основе GitHub Pages

## Использование

### Вариант 1: OCI-реестр (рекомендуется)

Скопируйте `release.yaml` в `.github/workflows/release.yaml`:

```bash
mkdir -p .github/workflows
cp release.yaml .github/workflows/
```

Установка чартов из OCI:

```bash
helm install my-operator oci://ghcr.io/YOUR_USERNAME/charts/postgres-operator --version 0.1.0
```

### Вариант 2: репозиторий Helm на GitHub Pages

Скопируйте `helm-gh-pages.yaml` в `.github/workflows/`:

```bash
cp helm-gh-pages.yaml .github/workflows/
```

Добавьте репозиторий Helm:

```bash
helm repo add postgres-operator https://YOUR_USERNAME.github.io/postgres-operator
helm repo update
helm install my-operator postgres-operator/postgres-operator
```

## Необходимые цели Makefile

Убедитесь, что в вашем Makefile есть эти цели:

```makefile
CHART_NAME ?= postgres-operator
CHART_VERSION ?= 0.1.0
CHART_DIR ?= charts/$(CHART_NAME)

.PHONY: helm-chart
helm-chart: manifests kustomize
	@mkdir -p $(CHART_DIR)/templates
	@cat > $(CHART_DIR)/Chart.yaml <<EOF
apiVersion: v2
name: $(CHART_NAME)
description: A Helm chart for $(CHART_NAME)
type: application
version: $(CHART_VERSION)
appVersion: "$(VERSION)"
EOF
	@cat > $(CHART_DIR)/values.yaml <<EOF
image:
  repository: $(IMAGE_TAG_BASE)
  tag: $(VERSION)
  pullPolicy: IfNotPresent
replicaCount: 1
EOF
	@cd config/manager && $(KUSTOMIZE) edit set image controller=$(IMG)
	@$(KUSTOMIZE) build config/default > $(CHART_DIR)/templates/manifests.yaml

.PHONY: helm-package
helm-package: helm-chart
	@mkdir -p dist
	helm package $(CHART_DIR) -d dist/

.PHONY: helm-lint
helm-lint: helm-chart
	helm lint $(CHART_DIR)
```

## Создание релиза

```bash
# Ensure you're on main branch
git checkout main

# Tag the release
git tag v0.1.0

# Push tag to trigger workflow
git push origin v0.1.0
```

## Разрешения

Workflows требуют следующих разрешений репозитория:

- `contents: write` — создание релизов, публикация в gh-pages
- `packages: write` — публикация в GHCR

Они настроены в файлах workflow с помощью блоков `permissions:`.

## Секреты

При использовании `GITHUB_TOKEN` дополнительные секреты не требуются. Workflows используют автоматический `secrets.GITHUB_TOKEN`, у которого есть подходящие разрешения для GHCR.
