---
layout: default
title: "Lab 07.3: High Availability"
nav_order: 13
parent: "Модуль 7: Подготовка к продакшену"
grand_parent: Модули
mermaid: true
---

# Лабораторная 7.3: Реализация высокой доступности

**Связанный урок:** [Урок 7.3: Высокая доступность](../lessons/03-high-availability.md)  
**Навигация:** [← Предыдущая лабораторная: RBAC](lab-02-rbac-security.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Производительность →](lab-04-performance-scalability.md)

## Цели

- Включить выбор лидера
- Развернуть несколько реплик
- Настроить лимиты ресурсов
- Протестировать сценарии отказоустойчивости (failover)
- Настроить бюджет прерывания подов (Pod Disruption Budget)

## Предварительные требования

- Завершение [Лабораторной 7.2](lab-02-rbac-security.md)
- Оператор, готовый к развёртыванию
- Понимание выбора лидера

## Упражнение 1: включение выбора лидера

Сгенерированный kubebuilder `cmd/main.go` уже поддерживает выбор лидера через флаг `--leader-elect`.

### Задача 1.1: изучите код выбора лидера

```bash
# Navigate to your operator project
cd ~/postgres-operator

# Review the leader election setup in main.go
grep -A 20 "LeaderElection" cmd/main.go
```

Вы должны увидеть код вроде:

```go
var enableLeaderElection bool
flag.BoolVar(&enableLeaderElection, "leader-elect", false,
    "Enable leader election for controller manager.")

mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    // ... other options ...
    LeaderElection:         enableLeaderElection,
    LeaderElectionID:       "your-operator-leader-election",
})
```

### Задача 1.2: включите выбор лидера в развёртывании

Обновите `config/manager/manager.yaml`, добавив флаг `--leader-elect`:

```yaml
spec:
  template:
    spec:
      containers:
      - name: manager
        args:
        - --leader-elect
        - --health-probe-bind-address=:8081
```

### Задача 1.3: разверните и проверьте

```bash
# For Docker: Build and Deploy the operator with network policies enabled
make docker-build IMG=postgres-operator:latest
kind load docker-image postgres-operator:latest --name k8s-operators-course
make deploy IMG=postgres-operator:latest

# For Podman: Build and Deploy operator - use localhost/ prefix to match the loaded image
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar
make deploy IMG=localhost/postgres-operator:latest

# Check for lease object
kubectl get lease -n postgres-operator-system

# Check logs for leader election
kubectl logs -n postgres-operator-system -l control-plane=controller-manager | grep -i "leader"
```

## Упражнение 2: развёртывание нескольких реплик

### Задача 2.1: обновите количество реплик развёртывания

Отредактируйте `config/manager/manager.yaml`, чтобы увеличить число реплик:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controller-manager
  namespace: system
spec:
  replicas: 3  # Change from 1 to 3
  selector:
    matchLabels:
      control-plane: controller-manager
  template:
    spec:
      containers:
      - name: manager
        args:
        - --leader-elect  # Required for HA
        - --health-probe-bind-address=:8081
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
          requests:
            cpu: 10m
            memory: 64Mi
```

### Задача 2.2: разверните и проверьте

```bash
# For Docker: Build and Deploy the operator with network policies enabled
make docker-build IMG=postgres-operator:latest
kind load docker-image postgres-operator:latest --name k8s-operators-course
make deploy IMG=postgres-operator:latest

# For Podman: Build and Deploy operator - use localhost/ prefix to match the loaded image
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar
make deploy IMG=localhost/postgres-operator:latest

# Check replicas
kubectl get deployment -n postgres-operator-system

# Check all pods are running
kubectl get pods -n postgres-operator-system -l control-plane=controller-manager

# Verify only one is leader (check logs)
for pod in $(kubectl get pods -n postgres-operator-system -l control-plane=controller-manager -o name); do
  echo "=== $pod ==="
  kubectl logs -n postgres-operator-system $pod | grep -i "leader" | tail -2
done
```

## Упражнение 3: настройка лимитов ресурсов

### Задача 3.1: задайте запросы и лимиты ресурсов

Обновите развёртывание:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Задача 3.2: отслеживайте использование ресурсов

```bash
# Check resource usage
kubectl top pods -l control-plane=controller-manager

# Watch resource usage
watch kubectl top pods -l control-plane=controller-manager
```

## Упражнение 4: тестирование отказоустойчивости

### Задача 4.1: определите лидера

```bash
# List all pods
kubectl get pods -n postgres-operator-system -l control-plane=controller-manager

# Find the lease and identify the leader
kubectl get lease -n postgres-operator-system -o yaml

# The holderIdentity field shows which pod is the leader
# Look for the pod name in the holderIdentity

# Check logs to confirm leader
LEADER_POD=$(kubectl get lease -n postgres-operator-system -o jsonpath='{.items[0].spec.holderIdentity}' | cut -d'_' -f1)
echo "Leader pod: $LEADER_POD"
kubectl logs -n postgres-operator-system $LEADER_POD | grep -i "became leader"
```

### Задача 4.2: имитируйте отказ лидера

```bash
# Get the leader pod name
LEADER_POD=$(kubectl get lease -n postgres-operator-system -o jsonpath='{.items[0].spec.holderIdentity}' | cut -d'_' -f1)

# Delete the leader pod
kubectl delete pod -n postgres-operator-system $LEADER_POD

# Watch failover happen
watch kubectl get pods -n postgres-operator-system -l control-plane=controller-manager

# In another terminal, watch the lease
watch kubectl get lease -n postgres-operator-system -o jsonpath='{.items[0].spec.holderIdentity}'

# After a new leader is elected, verify reconciliation continues
kubectl logs -n postgres-operator-system -l control-plane=controller-manager --tail=20 | grep -i "reconcil"
```

## Упражнение 5: бюджет прерывания подов (Pod Disruption Budget)

### Задача 5.1: создайте PDB

Создайте `config/manager/pdb.yaml`:

```bash
cat > config/manager/pdb.yaml << 'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: controller-manager-pdb
  namespace: system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      control-plane: controller-manager
EOF
```

Добавьте в `config/manager/kustomization.yaml`:

```yaml
resources:
- manager.yaml
- pdb.yaml
```

### Задача 5.2: разверните и протестируйте PDB

```bash
# For Docker: Build and Deploy the operator with network policies enabled
make docker-build IMG=postgres-operator:latest
kind load docker-image postgres-operator:latest --name k8s-operators-course
make deploy IMG=postgres-operator:latest

# For Podman: Build and Deploy operator - use localhost/ prefix to match the loaded image
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar
make deploy IMG=localhost/postgres-operator:latest

# Verify PDB is created
kubectl get pdb -n postgres-operator-system

# Check PDB status
kubectl describe pdb -n postgres-operator-system postgres-operator-controller-manager-pdb
```

**Важно:** PDB защищает только от **добровольных прерываний** (evictions), а НЕ от прямых команд `kubectl delete pod`!

### Задача 5.3: протестируйте PDB с помощью rollout restart

Проще всего протестировать PDB с помощью `kubectl rollout restart`, который внутри использует API вытеснения (eviction):

```bash
# Check current PDB status - note ALLOWED DISRUPTIONS
kubectl get pdb -n postgres-operator-system

# Expected output:
# NAME                                        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# postgres-operator-controller-manager-pdb   2               N/A               1                     5m

# Trigger a rolling restart (this respects PDB)
kubectl rollout restart deployment/postgres-operator-controller-manager -n postgres-operator-system

# Watch the rollout - PDB ensures at least 2 pods remain available
kubectl get pods -n postgres-operator-system -l control-plane=controller-manager -w

# In another terminal, watch PDB status during rollout
watch kubectl get pdb -n postgres-operator-system
```

**Что вы должны увидеть:**
- Поды заменяются по одному (не все сразу)
- `ALLOWED DISRUPTIONS` меняется по мере завершения/создания подов
- Как минимум 2 пода остаются в состоянии `Running` на протяжении всего процесса

### Задача 5.4: разберитесь, как PDB работает с обновлениями

Разберём математику, лежащую в основе PDB:

```bash
# Check current state
kubectl get pdb -n postgres-operator-system

# Formula: ALLOWED DISRUPTIONS = currentHealthy - minAvailable
# With 3 healthy pods and minAvailable=2: 3 - 2 = 1 disruption allowed
```

**Важно:** PDB НЕ блокирует обновления! Вот почему:

1. Начально: 3 здоровых пода, `minAvailable=2`, `allowedDisruptions=1`
2. Начинается обновление: создаётся новый под → 4 здоровых
3. `allowedDisruptions = 4 - 2 = 2` → старый под завершается
4. Теперь 3 здоровых (2 старых + 1 новый), `allowedDisruptions = 1`
5. Создаётся ещё один новый под → 4 здоровых → старый под завершается
6. Повторяется до завершения

**PDB обеспечивает замену подов ПО ОДНОМУ, а не всех сразу!**

### От чего PDB на самом деле защищает

PDB защищает от **внешних прерываний**, а не от обновлений развёртывания:

```bash
# PDB protects against these scenarios:

# 1. Node drain (cluster maintenance)
kubectl drain <node-name> --ignore-daemonsets
# PDB prevents draining if it would violate minAvailable

# 2. Cluster Autoscaler scale-down
# Autoscaler won't remove a node if it would violate PDB

# 3. Pod eviction due to resource pressure
# Kubelet respects PDB when evicting pods

# 4. Manual eviction API calls
# Tools using eviction API respect PDB
```

### Убедитесь, что PDB ограничивает частоту прерываний

Понаблюдайте за обновлением, чтобы увидеть, как PDB обеспечивает замену подов по одному:

```bash
# Ensure we have 3 replicas and minAvailable=2
kubectl scale deployment/postgres-operator-controller-manager -n postgres-operator-system --replicas=3
kubectl patch pdb postgres-operator-controller-manager-pdb -n postgres-operator-system \
  --type='json' -p='[{"op": "replace", "path": "/spec/minAvailable", "value": 2}]'

# Wait for stable state
sleep 10

# Watch pods during rollout - notice they're replaced ONE at a time
kubectl get pods -n postgres-operator-system -l control-plane=controller-manager -w &

# Trigger rollout
kubectl rollout restart deployment/postgres-operator-controller-manager -n postgres-operator-system

# Watch the rollout - pods replaced sequentially, not all at once
# Press Ctrl+C when done watching
```

**Без PDB** Kubernetes может завершить несколько подов одновременно во время прерываний. **С PDB** он гарантирует, что `minAvailable` подов всегда остаются запущенными.

### Понимание поведения PDB

```bash
# Check current PDB status
kubectl get pdb -n postgres-operator-system

# The columns mean:
# MIN AVAILABLE: Minimum pods that must remain running
# ALLOWED DISRUPTIONS: How many pods can be evicted right now
#
# Formula: ALLOWED DISRUPTIONS = currentHealthy - minAvailable
# Example: 3 healthy - 2 minimum = 1 allowed disruption
```

**Почему `kubectl delete pod` не соблюдает PDB:**
- `kubectl delete` — это **прямое удаление**, а не вытеснение (eviction)
- PDB защищает только от **API вытеснения (Eviction API)**, используемого:
  - `kubectl drain` (обслуживание узла)
  - `kubectl rollout restart` (обновления развёртывания)
  - Cluster Autoscaler (уменьшение масштаба)
  - Планировщик Kubernetes (вытеснение подов, preemption)
- В продакшене эти инструменты используют вытеснение, поэтому PDB работает как задумано

## Очистка

```bash
# Undeploy operator
make undeploy

# Or scale down for testing
kubectl scale deployment -n postgres-operator-system controller-manager --replicas=1
```

## Итоги лабораторной

В этой лабораторной вы:
- Включили выбор лидера через флаг `--leader-elect`
- Развернули несколько реплик, обновив `config/manager/manager.yaml`
- Настроили лимиты ресурсов
- Протестировали отказоустойчивость, удалив под-лидер
- Настроили бюджет прерывания подов
- Протестировали PDB с помощью API вытеснения

## Ключевые уроки

1. Выбор лидера включается флагом командной строки в kubebuilder
2. Увеличивайте число реплик в `config/manager/manager.yaml` для HA
3. Используйте `make deploy` для применения всех конфигураций
4. Отказоустойчивость автоматическая — резервные поды получают аренду (lease)
5. **PDB защищает только от добровольных прерываний** (вытеснений, а не прямого удаления)
6. Используйте `kubectl drain` или API вытеснения для тестирования PDB — НЕ `kubectl delete pod`
7. Проверки здоровья предварительно настроены kubebuilder

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Leader Election Configuration](../solutions/leader-election.go) — полная настройка выбора лидера
- [HA Deployment](../solutions/ha-deployment.yaml) — HA-развёртывание с PDB

## Дальнейшие шаги

Теперь давайте оптимизируем производительность!

**Навигация:** [← Предыдущая лабораторная: RBAC](lab-02-rbac-security.md) | [Связанный урок](../lessons/03-high-availability.md) | [Следующая лабораторная: Производительность →](lab-04-performance-scalability.md)
