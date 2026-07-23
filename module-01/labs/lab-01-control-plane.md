---
layout: default
title: "Lab 01.1: Control Plane"
nav_order: 11
parent: "Модуль 1: Архитектура Kubernetes"
grand_parent: Модули
mermaid: true
---

# Лабораторная 1.1: Исследование управляющего слоя

**Связанный урок:** [Урок 1.1: Обзор управляющего слоя Kubernetes](../lessons/01-control-plane.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: Механизмы API →](lab-02-api-machinery.md)

## Цели

- Исследовать компоненты управляющего слоя Kubernetes
- Понять, как компоненты взаимодействуют
- Наблюдать за поведением контроллера в реальном времени
- Отследить потоки запросов к API

## Предварительные требования

- Запущенный кластер kind
- Настроенный и работающий kubectl

## Упражнение 1: осмотр компонентов управляющего слоя

### Задача 1.1: просмотр подов управляющего слоя

```bash
# List all control plane components
kubectl get pods -n kube-system

# Get detailed information about API server
kubectl get pods -n kube-system -l component=kube-apiserver -o yaml | head -50

# Check controller manager
kubectl get pods -n kube-system -l component=kube-controller-manager

# Check scheduler
kubectl get pods -n kube-system -l component=kube-scheduler
```

**Ожидаемый результат**: вы должны увидеть поды API server, controller manager, scheduler и etcd.

### Задача 1.2: исследование API Server

```bash
# Get cluster information
kubectl cluster-info

# Get API server version
kubectl version --output=yaml

# Discover available API groups
kubectl api-versions | head -20

# Get API resources
kubectl api-resources | grep -E "NAME|deployments|pods|services"
```

**Вопросы для ответа:**
1. Какая версия Kubernetes запущена?
2. Сколько групп API доступно?
3. Какую версию API используют Deployments?

## Упражнение 2: наблюдение за поведением контроллера

### Задача 2.1: создайте и понаблюдайте за Deployment

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx:latest --replicas=3

# Immediately watch the deployment
kubectl get deployment nginx -w &
DEPLOY_PID=$!

# In another terminal (or wait a moment), watch ReplicaSets
kubectl get replicasets -w &
RS_PID=$!

# Watch pods
kubectl get pods -l app=nginx -w &
POD_PID=$!

# Wait 30 seconds to observe the creation flow
sleep 30

# Stop watching
kill $DEPLOY_PID $RS_PID $POD_PID 2>/dev/null
```

**Наблюдения:**
1. В каком порядке создавались ресурсы?
2. Сколько времени потребовалось, чтобы все поды стали готовы?
3. Какие поля status менялись во время создания?

### Задача 2.2: трассировка создания ресурсов

```bash
# Get the deployment with all details
kubectl get deployment nginx -o yaml > /tmp/nginx-deployment.yaml

# Get the ReplicaSet
kubectl get replicasets -l app=nginx -o yaml > /tmp/nginx-rs.yaml

# Get one of the pods
kubectl get pods -l app=nginx -o yaml | head -100 > /tmp/nginx-pod.yaml

# Examine the owner references
grep -A 5 "ownerReferences" /tmp/nginx-pod.yaml
grep -A 5 "ownerReferences" /tmp/nginx-rs.yaml
```

**Вопросы:**
1. Какова связь между Deployment, ReplicaSet и Pod?
2. Как используются ссылки-владельцы (owner references)?

## Упражнение 3: просмотр логов контроллера

### Задача 3.1: логи Controller Manager

```bash
# View recent controller manager logs
kubectl logs -n kube-system -l component=kube-controller-manager --tail=50

# Filter for deployment-related logs
kubectl logs -n kube-system -l component=kube-controller-manager --tail=100 | grep -i deployment

# Watch logs in real-time
kubectl logs -n kube-system -l component=kube-controller-manager -f --tail=20
```

**В другом терминале запустите действие:**
```bash
# Scale the deployment
kubectl scale deployment nginx --replicas=5

# Watch the logs to see controller activity
```

### Задача 3.2: логи Scheduler

```bash
# View scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler --tail=50

# Look for scheduling decisions
kubectl logs -n kube-system -l component=kube-scheduler --tail=100 | grep -i "scheduled"
```

## Упражнение 4: прямое взаимодействие с API

### Задача 4.1: используйте kubectl proxy

```bash
# Start kubectl proxy in background
kubectl proxy --port=8001 &
PROXY_PID=$!

# Wait for proxy to start
sleep 2

# Make direct API calls
curl http://localhost:8001/api/v1/namespaces

# Get pods via API
curl http://localhost:8001/api/v1/namespaces/default/pods | jq '.items[].metadata.name' | head -5

# Get the nginx deployment
curl http://localhost:8001/apis/apps/v1/namespaces/default/deployments/nginx | jq '.spec.replicas'

# Stop the proxy
kill $PROXY_PID
```

### Задача 4.2: создайте ресурс через API

```bash
# Start proxy again
kubectl proxy --port=8001 &
PROXY_PID=$!
sleep 2

# Create a pod via direct API call
curl -X POST http://localhost:8001/api/v1/namespaces/default/pods \
  -H "Content-Type: application/json" \
  -d '{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "api-created-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [{
        "name": "nginx",
        "image": "nginx:latest"
      }]
    }
  }'

# Verify it was created
kubectl get pod api-created-pod

# Clean up
kubectl delete pod api-created-pod
kill $PROXY_PID
```

## Упражнение 5: наблюдение за согласованием

### Задача 5.1: ручное удаление пода

```bash
# Get a pod name
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')

# Delete the pod
kubectl delete pod $POD_NAME

# Immediately watch for recreation
kubectl get pods -l app=nginx -w

# The ReplicaSet controller should recreate it!
```

### Задача 5.2: изменение желаемого состояния

```bash
# Scale down
kubectl scale deployment nginx --replicas=2

# Watch pods being terminated
kubectl get pods -l app=nginx -w

# Scale up
kubectl scale deployment nginx --replicas=4

# Watch pods being created
kubectl get pods -l app=nginx -w
```

## Очистка

```bash
# Delete the deployment (this will cascade delete ReplicaSet and Pods)
kubectl delete deployment nginx
```

## Итоги лабораторной

В этой лабораторной вы:
- Исследовали компоненты управляющего слоя
- Наблюдали за поведением контроллера в реальном времени
- Отследили потоки запросов к API
- Разобрались в процессе согласования
- Напрямую взаимодействовали с API Kubernetes

## Ключевые уроки

1. Компоненты управляющего слоя совместно управляют кластером
2. Контроллеры непрерывно отслеживают и согласовывают ресурсы
3. API Server — это центральный узел взаимодействия
4. Ссылки-владельцы (owner references) поддерживают связи между ресурсами
5. Согласование происходит автоматически, когда желаемое состояние не совпадает с фактическим

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-control-plane.md) | [Следующая лабораторная: Механизмы API →](lab-02-api-machinery.md)
