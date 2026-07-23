---
layout: default
title: "Lab 05.4: Webhook Deployment"
nav_order: 14
parent: "Модуль 5: Вебхуки и контроль допуска"
grand_parent: Модули
mermaid: true
---

# Лабораторная 5.4: Развёртывание вебхуков и сертификаты

**Связанный урок:** [Урок 5.4: Развёртывание вебхуков и сертификаты](../lessons/04-webhook-deployment.md)  
**Навигация:** [← Предыдущая лабораторная: Мутирующие вебхуки](lab-03-mutating-webhooks.md) | [Обзор модуля](../README.md)

## Цели

- Понять требования к сертификатам для вебхуков
- Развернуть оператор с вебхуками в кластер
- Настроить cert-manager для управления сертификатами
- Устранять неполадки вебхуков

## Предварительные требования

- Завершение [Лабораторной 5.3](lab-03-mutating-webhooks.md)
- Оператор Database с вебхуками
- Понимание TLS и сертификатов
- Кластер kind с установленным cert-manager (из `scripts/setup-kind-cluster.sh`)

## Понимание требований к сертификатам вебхуков

Вебхукам нужны TLS-сертификаты, потому что API-сервер Kubernetes взаимодействует с вебхуками по HTTPS. Сертификат должен быть доверенным для API-сервера.

**Два подхода:**
1. **cert-manager (рекомендуется)** — автоматически управляет сертификатами в кластере
2. **Ручные сертификаты** — только для особых случаев

> **Примечание:** проект, сгенерированный kubebuilder, уже настроен для работы с cert-manager. `config/default/kustomization.yaml` включает ресурсы cert-manager.

## Упражнение 1: проверка настройки Cert-Manager

### Задача 1.1: проверьте, что Cert-Manager запущен

```bash
# If you used scripts/setup-kind-cluster.sh, cert-manager is already installed
kubectl get pods -n cert-manager

# Should show:
# cert-manager-xxx          Running
# cert-manager-cainjector-xxx   Running
# cert-manager-webhook-xxx      Running
```

### Задача 1.2: установите Cert-Manager (если не установлен)

```bash
# Only if cert-manager is not running
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=120s
kubectl wait --for=condition=Available deployment/cert-manager-webhook -n cert-manager --timeout=120s
kubectl wait --for=condition=Available deployment/cert-manager-cainjector -n cert-manager --timeout=120s
```

## Упражнение 2: изучение конфигурации Cert-Manager в Kubebuilder

### Задача 2.1: проверьте конфигурацию сертификата

```bash
cd ~/postgres-operator

# Check cert-manager configuration
cat config/certmanager/certificate-*.yaml
```

Это определяет ресурс Certificate, который cert-manager будет использовать для генерации TLS-сертификатов для вебхука.

### Задача 2.2: проверьте Kustomization

```bash
# Check how cert-manager is integrated
cat config/default/kustomization.yaml
```

`config/default/kustomization.yaml` должен включать `../certmanager` для активации интеграции с cert-manager.

## Упражнение 3: развёртывание оператора с вебхуками

### Задача 3.1: соберите образ оператора

```bash
cd ~/postgres-operator

# Build the container image
make docker-build IMG=postgres-operator:latest

# For Podman users:
# make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
```

### Задача 3.2: загрузите образ в Kind

```bash
# For Docker:
kind load docker-image postgres-operator:latest --name k8s-operators-course

# For Podman:
# podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
# kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
# rm /tmp/postgres-operator.tar
```

### Задача 3.3: разверните в кластер

```bash
# Deploy operator with webhooks
make deploy IMG=postgres-operator:latest

# For Podman users:
# make deploy IMG=localhost/postgres-operator:latest
```

### Задача 3.4: проверьте развёртывание

```bash
# Check deployment
kubectl get deployment -n postgres-operator-system

# Check pods
kubectl get pods -n postgres-operator-system

# Wait for pod to be ready
kubectl wait --for=condition=Ready pod -l control-plane=controller-manager -n postgres-operator-system --timeout=120s
```

## Упражнение 4: проверка конфигурации вебхука

### Задача 4.1: проверьте конфигурации вебхуков

```bash
# Check validating webhook
kubectl get validatingwebhookconfigurations

# Check mutating webhook  
kubectl get mutatingwebhookconfigurations

# Get details
kubectl describe validatingwebhookconfiguration postgres-operator-validating-webhook-configuration
```

### Задача 4.2: проверьте сертификаты

```bash
# Check certificate was created by cert-manager
kubectl get certificate -n postgres-operator-system

# Check certificate status
kubectl describe certificate -n postgres-operator-system

# Check the secret containing TLS certs
kubectl get secret -n postgres-operator-system | grep tls
```

### Задача 4.3: проверьте сервис вебхука

```bash
# Check webhook service
kubectl get service -n postgres-operator-system

# Check service endpoints
kubectl get endpoints -n postgres-operator-system
```

## Упражнение 5: тестирование вебхуков

### Задача 5.1: протестируйте мутирующий вебхук (значения по умолчанию)

```bash
# Create resource with minimal spec
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: webhook-test
spec:
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Check if defaults were applied
echo "Image (should be defaulted):"
kubectl get database webhook-test -o jsonpath='{.spec.image}'
echo

echo "Replicas (should be defaulted):"
kubectl get database webhook-test -o jsonpath='{.spec.replicas}'
echo

echo "Labels (should include managed-by):"
kubectl get database webhook-test -o jsonpath='{.metadata.labels}'
echo
```

### Задача 5.2: протестируйте валидирующий вебхук (отклонение)

```bash
# Try to create invalid resource
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-test
spec:
  image: nginx:latest  # Invalid - not a postgres image
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Should be rejected with validation error
```

### Задача 5.3: протестируйте валидацию обновления

```bash
# Try to reduce storage (should fail)
kubectl patch database webhook-test --type merge -p '{"spec":{"storage":{"size":"5Gi"}}}'

# Should be rejected
```

## Упражнение 6: устранение неполадок вебхуков

### Задача 6.1: проверьте логи оператора

```bash
# Get operator logs
kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager

# Look for webhook-related messages
kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager | grep -i webhook
```

### Задача 6.2: проверьте статус сертификата

```bash
# Check certificate status
kubectl get certificate -n postgres-operator-system -o wide

# Describe for details
kubectl describe certificate -n postgres-operator-system

# Check cert-manager logs if certificate not ready
kubectl logs -n cert-manager deployment/cert-manager
```

### Задача 6.3: распространённые проблемы и их решения

**Проблема: сертификат не готов**
```bash
# Check cert-manager is running
kubectl get pods -n cert-manager

# Check certificate events
kubectl describe certificate -n postgres-operator-system
```

**Проблема: отказ в соединении с вебхуком**
```bash
# Check service endpoints
kubectl get endpoints -n postgres-operator-system

# Check pod is running
kubectl get pods -n postgres-operator-system
```

**Проблема: несоответствие CA bundle**
```bash
# Check CA bundle in webhook config
kubectl get validatingwebhookconfiguration -o jsonpath='{.items[0].webhooks[0].clientConfig.caBundle}' | base64 -d | openssl x509 -text -noout | head -20
```

## Очистка

```bash
# Delete test resources
kubectl delete databases --all

# Undeploy operator
make undeploy
```

## Итоги лабораторной

В этой лабораторной вы:
- Проверили настройку cert-manager
- Изучили интеграцию kubebuilder с cert-manager
- Развернули оператор с вебхуками в кластер
- Проверили конфигурацию вебхуков и сертификаты
- Протестировали функциональность вебхуков
- Изучили приёмы устранения неполадок

## Ключевые уроки

1. Вебхукам нужны TLS-сертификаты — API-сервер должен им доверять
2. cert-manager автоматически управляет сертификатами в кластере
3. Проекты kubebuilder предварительно настроены для cert-manager
4. Вебхуки не могут легко работать через `make run` (требуют развёртывания в кластере)
5. Используйте `kubectl describe` и логи для устранения неполадок
6. Проблемы с сертификатами распространены — сначала проверяйте статус cert-manager

## Решения

Эта лабораторная посвящена развёртыванию и сертификатам. Для реализации вебхуков см.:
- [Validating Webhook](../solutions/validating-webhook.go) — из Лабораторной 5.2
- [Mutating Webhook](../solutions/mutating-webhook.go) — из Лабораторной 5.3

## Поздравляем!

Вы завершили Модуль 5! Теперь вы понимаете:
- Контроль допуска и вебхуки
- Валидирующие вебхуки для пользовательской валидации
- Мутирующие вебхуки для установки значений по умолчанию
- Управление сертификатами с cert-manager
- Развёртывание вебхуков и устранение неполадок

В Модуле 6 вы изучите тестирование и отладку операторов!

**Навигация:** [← Предыдущая лабораторная: Мутирующие вебхуки](lab-03-mutating-webhooks.md) | [Связанный урок](../lessons/04-webhook-deployment.md) | [Обзор модуля](../README.md)
