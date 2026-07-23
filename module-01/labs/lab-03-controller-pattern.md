---
layout: default
title: "Lab 01.3: Controller Pattern"
nav_order: 13
parent: "Модуль 1: Архитектура Kubernetes"
grand_parent: Модули
mermaid: true
---

# Лабораторная 1.3: Наблюдение за контроллерами в действии

**Связанный урок:** [Урок 1.3: Паттерн контроллера](../lessons/03-controller-pattern.md)  
**Навигация:** [← Предыдущая лабораторная: Механизмы API](lab-02-api-machinery.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Пользовательские ресурсы →](lab-04-custom-resources.md)

## Цели

- Наблюдать за согласованием контроллера в реальном времени
- Понять паттерн цикла управления (control loop)
- Увидеть декларативное и императивное поведение
- Проверить идемпотентность
- Разобраться в механизмах отслеживания (watch)

## Предварительные требования

- Запущенный кластер kind
- Настроенный kubectl

## Упражнение 1: наблюдение за циклом согласования

### Задача 1.1: создайте и понаблюдайте за Deployment

```bash
# Create a deployment
kubectl create deployment controller-demo --image=nginx:latest --replicas=2

# Watch deployment in one terminal
kubectl get deployment controller-demo -w

# In another terminal, watch ReplicaSet
kubectl get replicasets -w

# In another terminal, watch pods
kubectl get pods -l app=controller-demo -w
```

**Наблюдения:**
1. Что создалось первым?
2. Сколько времени прошло, пока все ресурсы стали готовы?
3. Какие поля status менялись?

### Задача 1.2: отследите согласование

```bash
# Get events to see the flow
kubectl get events --sort-by='.lastTimestamp' | grep controller-demo

# Get detailed deployment info
kubectl get deployment controller-demo -o yaml | grep -A 10 status:

# Check ReplicaSet owner reference
kubectl get replicasets -l app=controller-demo -o yaml | grep -A 10 ownerReferences

# Check Pod owner references
kubectl get pods -l app=controller-demo -o yaml | grep -A 10 ownerReferences
```

## Упражнение 2: проверка согласования

### Задача 2.1: ручное удаление пода

```bash
# Get a pod name
POD_NAME=$(kubectl get pods -l app=controller-demo -o jsonpath='{.items[0].metadata.name}')
echo "Deleting pod: $POD_NAME"

# Delete the pod
kubectl delete pod $POD_NAME

# Immediately watch for recreation
echo "Watching for pod recreation..."
kubectl get pods -l app=controller-demo -w
```

**Вопросы:**
1. Как быстро под был пересоздан?
2. Какой контроллер его пересоздал?
3. Что это говорит вам о цикле управления?

### Задача 2.2: изменение желаемого состояния

```bash
# Scale up
kubectl scale deployment controller-demo --replicas=5

# Watch pods being created
kubectl get pods -l app=controller-demo -w

# Scale down
kubectl scale deployment controller-demo --replicas=1

# Watch pods being terminated
kubectl get pods -l app=controller-demo -w
```

**Наблюдения:**
1. Как контроллер обрабатывает увеличение масштаба?
2. Как он обрабатывает уменьшение?
3. Каков порядок операций?

## Упражнение 3: декларативное поведение

### Задача 3.1: примените один и тот же ресурс несколько раз

```bash
# Create a deployment manifest
cat <<EOF > /tmp/test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: declarative-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: declarative
  template:
    metadata:
      labels:
        app: declarative
    spec:
      containers:
      - name: nginx
        image: nginx:latest
EOF

# Apply it
kubectl apply -f /tmp/test-deployment.yaml

# Wait for it to be ready
kubectl wait --for=condition=available deployment/declarative-test --timeout=60s

# Count pods
kubectl get pods -l app=declarative | wc -l

# Apply the SAME file again
kubectl apply -f /tmp/test-deployment.yaml

# Count pods again (should be the same!)
kubectl get pods -l app=declarative | wc -l
```

**Ключевой вывод:** многократное применение одного и того же ресурса идемпотентно — оно не создаёт дубликатов.

### Задача 3.2: измените и примените заново

```bash
# Modify the manifest (change image)
cat <<EOF > /tmp/test-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: declarative-test
spec:
  replicas: 3
  selector:
    matchLabels:
      app: declarative
  template:
    metadata:
      labels:
        app: declarative
    spec:
      containers:
      - name: nginx
        image: nginx:1.21  # Changed from latest
EOF

# Apply the modified version
kubectl apply -f /tmp/test-deployment.yaml

# Watch the rolling update
kubectl get pods -l app=declarative -w
```

**Наблюдения:**
1. Что произошло, когда вы изменили образ?
2. Как Kubernetes обработал обновление?
3. Это декларативный подход — вы описали, что хотите, а Kubernetes сам определил, как этого достичь.

## Упражнение 4: логи контроллера

### Задача 4.1: просмотрите логи Controller Manager

```bash
# View recent logs
kubectl logs -n kube-system -l component=kube-controller-manager --tail=50

# Filter for our deployment
kubectl logs -n kube-system -l component=kube-controller-manager --tail=100 | grep declarative-test
```

### Задача 4.2: наблюдайте за логами во время действия

```bash
# Start watching logs in background
kubectl logs -n kube-system -l component=kube-controller-manager -f --tail=20 > /tmp/controller.log &
LOG_PID=$!

# Trigger an action
kubectl scale deployment declarative-test --replicas=5

# Wait a moment
sleep 5

# Check the logs
cat /tmp/controller.log | tail -20

# Stop log watching
kill $LOG_PID
```

## Упражнение 5: проверка идемпотентности

### Задача 5.1: многократное применение

```bash
# Apply the same deployment 5 times
for i in {1..5}; do
  echo "Apply #$i"
  kubectl apply -f /tmp/test-deployment.yaml
  sleep 2
done

# Check how many deployments exist
kubectl get deployments declarative-test

# Check how many ReplicaSets exist
kubectl get replicasets -l app=declarative

# Check how many pods exist
kubectl get pods -l app=declarative
```

**Ожидаемый результат:** только один Deployment, один ReplicaSet и корректное количество подов.

### Задача 5.2: подтвердите идемпотентность

```bash
# Get current state
kubectl get deployment declarative-test -o yaml > /tmp/before.yaml

# Apply again
kubectl apply -f /tmp/test-deployment.yaml

# Get state after
kubectl get deployment declarative-test -o yaml > /tmp/after.yaml

# Compare (they should be identical or very similar)
diff /tmp/before.yaml /tmp/after.yaml
```

## Упражнение 6: механизм отслеживания

### Задача 6.1: используйте kubectl watch

```bash
# Watch deployments
kubectl get deployments -w

# In another terminal, make changes
kubectl scale deployment declarative-test --replicas=2
kubectl scale deployment declarative-test --replicas=4
```

**Наблюдения:**
1. Как быстро вы видите обновления?
2. Какая информация отображается в выводе watch?

### Задача 6.2: наблюдайте за потоком событий

```bash
# Watch events
kubectl get events -w --sort-by='.lastTimestamp'

# In another terminal, trigger actions
kubectl scale deployment declarative-test --replicas=1
kubectl label deployment declarative-test env=test
```

## Упражнение 7: обновления статуса

### Задача 7.1: отслеживайте изменения статуса

```bash
# Watch status fields
watch -n 1 'kubectl get deployment declarative-test -o jsonpath="{.status.conditions[?(@.type==\"Available\")].status}"'

# In another terminal, scale
kubectl scale deployment declarative-test --replicas=0
kubectl scale deployment declarative-test --replicas=3
```

### Задача 7.2: сравните spec и status

```bash
# Get desired vs actual
echo "Desired replicas: $(kubectl get deployment declarative-test -o jsonpath='{.spec.replicas}')"
echo "Actual replicas: $(kubectl get deployment declarative-test -o jsonpath='{.status.replicas}')"
echo "Ready replicas: $(kubectl get deployment declarative-test -o jsonpath='{.status.readyReplicas}')"

# The controller continuously works to make actual match desired
```

## Очистка

```bash
# Delete deployments
kubectl delete deployment controller-demo
kubectl delete deployment declarative-test

# Clean up temp files
rm -f /tmp/test-deployment.yaml /tmp/before.yaml /tmp/after.yaml /tmp/controller.log
```

## Итоги лабораторной

В этой лабораторной вы:
- Наблюдали за согласованием контроллера в реальном времени
- Проверили декларативное поведение
- Подтвердили идемпотентность
- Разобрались в паттерне цикла управления
- Отслеживали обновления статуса

## Ключевые уроки

1. Контроллеры непрерывно согласовывают желаемое и фактическое состояние
2. Согласование происходит автоматически при изменении состояния
3. Kubernetes использует декларативную модель — описывайте, что вы хотите
4. Операции идемпотентны — их безопасно повторять
5. Поля status отражают фактическое состояние и обновляются контроллерами
6. Механизмы отслеживания обеспечивают обновления в реальном времени

**Навигация:** [← Предыдущая лабораторная: Механизмы API](lab-02-api-machinery.md) | [Связанный урок](../lessons/03-controller-pattern.md) | [Следующая лабораторная: Пользовательские ресурсы →](lab-04-custom-resources.md)
