---
layout: default
title: "Lab 02.3: Dev Environment"
nav_order: 13
parent: "Модуль 2: Введение в операторы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 2.3: Настройка вашей среды

**Связанный урок:** [Урок 2.3: Настройка среды разработки](../lessons/03-dev-environment.md)  
**Навигация:** [← Предыдущая лабораторная: Основы Kubebuilder](lab-02-kubebuilder-fundamentals.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Первый оператор →](lab-04-first-operator.md)

## Цели

- Проверить, что все необходимые инструменты установлены
- Настроить полноценную среду разработки
- Создать и проверить кластер kind
- Протестировать полную настройку

## Предварительные требования

- Завершение настройки [Модуля 1](../module-01/README.md)
- Базовое понимание необходимых инструментов

## Упражнение 1: проверка предварительных требований

### Задача 1.1: проверьте установку Go

```bash
# Check Go version (need 1.21+)
go version

# Verify Go is working
go env

# Check GOPATH and GOROOT
echo $GOPATH
echo $GOROOT
```

**Ожидается:** Go 1.21 или выше

### Задача 1.2: проверьте kubectl

```bash
# Check kubectl version
kubectl version --client

# Verify kubectl is working
kubectl cluster-info
```

**Примечание:** если кластер не настроен — это нормально, мы его создадим.

### Задача 1.3: проверьте Docker/Podman

```bash
# Check Docker
docker --version
docker info

# OR check Podman
podman --version
podman info
```

**Ожидается:** запущенный Docker или Podman

## Упражнение 2: установка недостающих инструментов

### Задача 2.1: используйте скрипт настройки

```bash
# Run the setup script
./scripts/setup-dev-environment.sh
```

**Обратите внимание:**
- Какие инструменты уже установлены?
- Какие инструменты нужно установить?
- Что устанавливается автоматически?

### Задача 2.2: ручная проверка

После запуска скрипта проверьте каждый инструмент:

```bash
# Go
go version

# kubectl
kubectl version --client

# kubebuilder
kubebuilder version

# kind
kind version

# Docker/Podman
docker --version  # or podman --version
```

## Упражнение 3: настройка кластера kind

### Задача 3.1: используйте скрипт настройки

```bash
# Run kind cluster setup
./scripts/setup-kind-cluster.sh
```

**Обратите внимание:**
- Процесс создания кластера
- Что устанавливается?
- Сколько времени это занимает?

### Задача 3.2: проверьте кластер

```bash
# Check cluster info
kubectl cluster-info

# List nodes
kubectl get nodes

# Check cluster context
kubectl config current-context

# Should show: kind-k8s-operators-course
```

### Задача 3.3: протестируйте кластер

```bash
# Create a test pod
kubectl run test-pod --image=nginx:latest

# Wait for it to be ready
kubectl wait --for=condition=ready pod/test-pod --timeout=60s

# Verify it's running
kubectl get pods

# Clean up
kubectl delete pod test-pod
```

## Упражнение 4: проверка kubebuilder

### Задача 4.1: проверьте установку

```bash
# Check kubebuilder version
kubebuilder version

# Verify it's in PATH
which kubebuilder

# Test kubebuilder commands
kubebuilder --help
```

### Задача 4.2: протестируйте kubebuilder init

```bash
# Create a test directory
mkdir -p /tmp/env-test
cd /tmp/env-test

# Test kubebuilder init
kubebuilder init --domain test.com --repo github.com/test/env-test

# Verify project was created
ls -la

# Check main.go exists
test -f main.go && echo "✅ main.go exists" || echo "❌ main.go missing"

# Clean up
cd ~
rm -rf /tmp/env-test
```

## Упражнение 5: полный чек-лист среды

### Задача 5.1: выполните чек-лист проверки

Проверьте каждый пункт:

```bash
# Go 1.21+
go version | grep -q "go1.2[1-9]\|go1.[3-9]" && echo "✅ Go version OK" || echo "❌ Go version too old"

# kubectl
kubectl version --client > /dev/null 2>&1 && echo "✅ kubectl OK" || echo "❌ kubectl missing"

# kubebuilder
kubebuilder version > /dev/null 2>&1 && echo "✅ kubebuilder OK" || echo "❌ kubebuilder missing"

# kind
kind version > /dev/null 2>&1 && echo "✅ kind OK" || echo "❌ kind missing"

# Docker/Podman
(docker info > /dev/null 2>&1 || podman info > /dev/null 2>&1) && echo "✅ Container runtime OK" || echo "❌ Container runtime missing"

# Kind cluster
kubectl cluster-info --context kind-k8s-operators-course > /dev/null 2>&1 && echo "✅ Kind cluster OK" || echo "❌ Kind cluster missing"
```

### Задача 5.2: устраните любые проблемы

Если какие-либо проверки не прошли:
- Просмотрите сообщения об ошибках
- Перезапустите скрипты настройки
- Проверьте документацию
- При необходимости обратитесь за помощью

## Упражнение 6: тестирование рабочего процесса разработки

### Задача 6.1: создайте тестовый проект

```bash
# Create test project
mkdir -p /tmp/workflow-test
cd /tmp/workflow-test

# Initialize project
kubebuilder init --domain test.com --repo github.com/test/workflow-test
```

### Задача 6.2: сгенерируйте и установите

```bash
# Generate code
make generate

# Generate manifests
make manifests

# Install CRD (this will fail if no cluster, that's OK for testing)
make install || echo "No cluster, skipping install"
```

### Задача 6.3: проверьте рабочий процесс

```bash
# Check generated files
ls -la config/crd/bases/
ls -la config/rbac/

# Check Makefile targets
make help
```

### Задача 6.4: очистка

```bash
cd ~
rm -rf /tmp/workflow-test
```

## Упражнение 7: настройка IDE (опционально)

### Задача 7.1: настройка VS Code

Если вы используете VS Code:

```bash
# Install Go extension
code --install-extension golang.go

# Install Kubernetes extension
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
```

### Задача 7.2: настройка GoLand

Если вы используете GoLand:
- В GoLand есть встроенная поддержка Go и Kubernetes
- Дополнительная настройка не нужна

## Сводка по проверке среды

В вашей среде должно быть:

- ✅ Go 1.21+
- ✅ kubectl
- ✅ kubebuilder
- ✅ kind
- ✅ Docker или Podman
- ✅ Запущенный кластер kind
- ✅ Контекст kubectl настроен на кластер kind

## Устранение неполадок

### Проблема: kubebuilder не найден
```bash
# Add to PATH
export PATH=$PATH:/usr/local/bin
# Or reinstall
```

### Проблема: кластер kind недоступен
```bash
# Recreate cluster
kind delete cluster --name k8s-operators-course
./scripts/setup-kind-cluster.sh
```

### Проблема: ошибки Go-модулей
```bash
# Enable Go modules
export GO111MODULE=on
```

## Итоги лабораторной

В этой лабораторной вы:
- Проверили все необходимые инструменты
- Настроили полноценную среду разработки
- Создали и проверили кластер kind
- Протестировали рабочий процесс разработки
- Убедились, что всё работает вместе

## Ключевые уроки

1. Полноценная среда включает: Go, kubebuilder, kubectl, kind, Docker/Podman
2. Скрипты настройки автоматизируют установку
3. Кластер kind предоставляет локальный Kubernetes
4. Все инструменты должны работать вместе
5. Проверка важна перед началом разработки

## Дальнейшие шаги

Ваша среда готова! Теперь давайте создадим ваш первый оператор.

**Навигация:** [← Предыдущая лабораторная: Основы Kubebuilder](lab-02-kubebuilder-fundamentals.md) | [Связанный урок](../lessons/03-dev-environment.md) | [Следующая лабораторная: Первый оператор →](lab-04-first-operator.md)
