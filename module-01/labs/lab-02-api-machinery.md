---
layout: default
title: "Lab 01.2: Api Machinery"
nav_order: 12
parent: "Модуль 1: Архитектура Kubernetes"
grand_parent: Модули
mermaid: true
---

# Лабораторная 1.2: Работа с API Kubernetes

**Связанный урок:** [Урок 1.2: Механизмы API Kubernetes](../lessons/02-api-machinery.md)  
**Навигация:** [← Предыдущая лабораторная: Управляющий слой](lab-01-control-plane.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Паттерн контроллера →](lab-03-controller-pattern.md)

## Цели

- Понять структуру API Kubernetes
- Обнаружить группы и версии API
- Выполнить прямые вызовы API
- Разобраться в структуре ресурса (spec против status)
- Поработать с версиями ресурсов

## Предварительные требования

- Запущенный кластер kind
- Настроенный kubectl
- Установленный `jq` (опционально, для разбора JSON)

## Упражнение 1: обнаружение API

### Задача 1.1: исследование версий API

```bash
# List all API versions
kubectl api-versions

# Count API groups
kubectl api-versions | cut -d'/' -f1 | sort -u | wc -l

# List core API group resources
kubectl api-versions | grep "^v1$"

# List apps API group
kubectl api-versions | grep "^apps/"
```

**Вопросы:**
1. Сколько существует групп API?
2. В чём разница между `/api/v1` и `/apis/apps/v1`?

### Задача 1.2: обнаружение ресурсов API

```bash
# List all API resources
kubectl api-resources

# List resources in core group
kubectl api-resources --api-group=""

# List resources in apps group
kubectl api-resources --api-group="apps"

# Get detailed information
kubectl api-resources -o wide | head -20
```

### Задача 1.3: изучение деталей группы API

```bash
# Get apps API group information
kubectl get --raw /apis/apps/v1 | jq '.'

# See what resources are available
kubectl get --raw /apis/apps/v1 | jq '.resources[].name'

# Get deployment API details
kubectl get --raw /apis/apps/v1 | jq '.resources[] | select(.name == "deployments")'
```

## Упражнение 2: структура ресурса

### Задача 2.1: изучение структуры ресурса

```bash
# Create a simple deployment
kubectl create deployment test-api --image=nginx:latest

# Get the deployment in YAML format
kubectl get deployment test-api -o yaml > /tmp/deployment.yaml

# Examine the structure
cat /tmp/deployment.yaml

# Extract just the spec
kubectl get deployment test-api -o jsonpath='{.spec}' | jq '.'

# Extract just the status
kubectl get deployment test-api -o jsonpath='{.status}' | jq '.'
```

**Наблюдения:**
1. Какие поля находятся в `spec`?
2. Какие поля находятся в `status`?
3. Чем они различаются?

### Задача 2.2: сравнение spec и status

```bash
# Get desired replicas (from spec)
kubectl get deployment test-api -o jsonpath='{.spec.replicas}'
echo

# Get actual replicas (from status)
kubectl get deployment test-api -o jsonpath='{.status.replicas}'
echo

# Get ready replicas
kubectl get deployment test-api -o jsonpath='{.status.readyReplicas}'
echo

# Wait for deployment to be ready, then check again
kubectl wait --for=condition=available deployment/test-api --timeout=60s
kubectl get deployment test-api -o jsonpath='{.spec.replicas}' && echo " (desired)"
kubectl get deployment test-api -o jsonpath='{.status.replicas}' && echo " (actual)"
kubectl get deployment test-api -o jsonpath='{.status.readyReplicas}' && echo " (ready)"
```

## Упражнение 3: прямые вызовы API

### Задача 3.1: запустите kubectl proxy

```bash
# Start proxy
kubectl proxy --port=8001 &
PROXY_PID=$!

# Wait for it to start
sleep 2

# Test connectivity
curl http://localhost:8001/api/v1
```

### Задача 3.2: получите список ресурсов через API

```bash
# List namespaces
curl -s http://localhost:8001/api/v1/namespaces | jq '.items[].metadata.name'

# List pods in default namespace
curl -s http://localhost:8001/api/v1/namespaces/default/pods | jq '.items[].metadata.name'

# List deployments
curl -s http://localhost:8001/apis/apps/v1/namespaces/default/deployments | jq '.items[].metadata.name'
```

### Задача 3.3: получите конкретный ресурс

```bash
# Get the test-api deployment
curl -s http://localhost:8001/apis/apps/v1/namespaces/default/deployments/test-api | jq '.metadata.name'
curl -s http://localhost:8001/apis/apps/v1/namespaces/default/deployments/test-api | jq '.spec.replicas'
curl -s http://localhost:8001/apis/apps/v1/namespaces/default/deployments/test-api | jq '.status'
```

### Задача 3.4: создайте ресурс через API

```bash
# Create a pod via API
curl -X POST http://localhost:8001/api/v1/namespaces/default/pods \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "api-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [{
        "name": "nginx",
        "image": "nginx:latest"
      }]
    }
  }' | jq '.metadata.name'

# Verify it was created
kubectl get pod api-pod
```

### Задача 3.5: обновите ресурс через API

```bash
# Get current resource version
RV=$(curl -s http://localhost:8001/api/v1/namespaces/default/pods/api-pod | jq -r '.metadata.resourceVersion')
echo "Current resourceVersion: $RV"

# Add a label
curl -X PATCH http://localhost:8001/api/v1/namespaces/default/pods/api-pod \
  -H "Content-Type: application/merge-patch+json" \
  -d '{
    "metadata": {
      "labels": {
        "created-by": "api"
      }
    }
  }' | jq '.metadata.labels'

# Verify the label
kubectl get pod api-pod --show-labels
```

## Упражнение 4: версии ресурсов

### Задача 4.1: разберитесь в версии ресурса

```bash
# Get resource version
kubectl get pod api-pod -o jsonpath='{.metadata.resourceVersion}'
echo

# Make a change
kubectl label pod api-pod test=value

# Get resource version again (should be different)
kubectl get pod api-pod -o jsonpath='{.metadata.resourceVersion}'
echo
```

### Задача 4.2: оптимистичное управление конкурентным доступом

```bash
# Get current resource version
RV1=$(kubectl get pod api-pod -o jsonpath='{.metadata.resourceVersion}')

# Try to update with old resource version (should fail or conflict)
curl -X PATCH http://localhost:8001/api/v1/namespaces/default/pods/api-pod \
  -H "Content-Type: application/merge-patch+json" \
  -d "{
    \"metadata\": {
      \"resourceVersion\": \"$RV1\",
      \"labels\": {
        \"test2\": \"value2\"
      }
    }
  }" | jq '.'

# Get new resource version
RV2=$(kubectl get pod api-pod -o jsonpath='{.metadata.resourceVersion}')

# Update with correct resource version
curl -X PATCH http://localhost:8001/api/v1/namespaces/default/pods/api-pod \
  -H "Content-Type: application/merge-patch+json" \
  -d "{
    \"metadata\": {
      \"resourceVersion\": \"$RV2\",
      \"labels\": {
        \"test2\": \"value2\"
      }
    }
  }" | jq '.metadata.labels'
```

## Упражнение 5: подресурсы

### Задача 5.1: подресурс status

```bash
# Get status subresource
curl -s http://localhost:8001/api/v1/namespaces/default/pods/api-pod/status | jq '.phase'

# Get scale subresource (for deployments)
curl -s http://localhost:8001/apis/apps/v1/namespaces/default/deployments/test-api/scale | jq '.'
```

### Задача 5.2: масштабирование через подресурс

```bash
# Get current scale
curl -s http://localhost:8001/apis/apps/v1/namespaces/default/deployments/test-api/scale | jq '.spec.replicas'

# Update scale
curl -X PATCH http://localhost:8001/apis/apps/v1/namespaces/default/deployments/test-api/scale \
  -H "Content-Type: application/merge-patch+json" \
  -d '{
    "spec": {
      "replicas": 3
    }
  }' | jq '.spec.replicas'

# Verify
kubectl get deployment test-api
```

## Очистка

```bash
# Stop proxy
kill $PROXY_PID 2>/dev/null

# Delete resources
kubectl delete deployment test-api
kubectl delete pod api-pod
```

## Итоги лабораторной

В этой лабораторной вы:
- Обнаружили группы и версии API
- Изучили структуру ресурса (spec против status)
- Выполнили прямые вызовы API с помощью kubectl proxy
- Разобрались в версиях ресурсов и оптимистичном управлении конкурентным доступом
- Поработали с подресурсами (status, scale)

## Ключевые уроки

1. API Kubernetes построен по принципам REST и организован в группы
2. Ресурсы имеют единообразную структуру: apiVersion, kind, metadata, spec, status
3. Spec описывает желаемое состояние, status — фактическое
4. Версии ресурсов обеспечивают оптимистичное управление конкурентным доступом
5. Подресурсы предоставляют дополнительную функциональность (status, scale, exec и т. д.)

**Навигация:** [← Предыдущая лабораторная: Управляющий слой](lab-01-control-plane.md) | [Связанный урок](../lessons/02-api-machinery.md) | [Следующая лабораторная: Паттерн контроллера →](lab-03-controller-pattern.md)
