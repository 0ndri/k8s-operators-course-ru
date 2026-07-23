---
layout: default
title: "Lab 07.4: Performance Scalability"
nav_order: 14
parent: "Модуль 7: Подготовка к продакшену"
grand_parent: Модули
mermaid: true
---

# Лабораторная 7.4: Оптимизация производительности

**Связанный урок:** [Урок 7.4: Производительность и масштабируемость](../lessons/04-performance-scalability.md)  
**Навигация:** [← Предыдущая лабораторная: HA](lab-03-high-availability.md) | [Обзор модуля](../README.md)

## Цели

- Реализовать ограничение частоты (rate limiting)
- Добавить стратегии кеширования
- Оптимизировать согласование
- Профилировать и оптимизировать производительность

## Предварительные требования

- Завершение [Лабораторной 7.3](lab-03-high-availability.md)
- Оператор с настройкой HA
- Понимание концепций производительности

## Упражнение 1: настройка ограничения частоты контроллера

У Controller-runtime (используемого kubebuilder) есть встроенное ограничение частоты. Настроим его.

### Задача 1.1: настройте MaxConcurrentReconciles

Обновите `SetupWithManager` вашего контроллера в `internal/controller/database_controller.go`:

```go
import (
    "time"
    
    "k8s.io/client-go/util/workqueue"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/controller"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Owns(&corev1.Secret{}).
        WithOptions(controller.Options{
            // Limit concurrent reconciliations
            MaxConcurrentReconciles: 2,
            // Custom rate limiter for requeue (typed for controller-runtime v0.19+)
            RateLimiter: workqueue.NewTypedItemExponentialFailureRateLimiter[reconcile.Request](
                time.Millisecond*5,    // Base delay
                time.Second*1000,      // Max delay
            ),
        }).
        Complete(r)
}
```

### Задача 1.2: добавьте ограничение частоты для внешних вызовов API (опционально)

Если ваш оператор вызывает внешние API, добавьте ограничение частоты:

```go
import (
    "golang.org/x/time/rate"
)

type DatabaseReconciler struct {
    client.Client
    Scheme     *runtime.Scheme
    APILimiter *rate.Limiter  // For external API calls
}

// In cmd/main.go when creating the reconciler:
if err = (&controller.DatabaseReconciler{
    Client:     mgr.GetClient(),
    Scheme:     mgr.GetScheme(),
    APILimiter: rate.NewLimiter(rate.Limit(10), 1), // 10 req/sec
}).SetupWithManager(mgr); err != nil {
    setupLog.Error(err, "unable to create controller", "controller", "Database")
    os.Exit(1)
}

// In Reconcile, use before external calls:
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Wait for rate limiter before external API calls
    if err := r.APILimiter.Wait(ctx); err != nil {
        return ctrl.Result{}, err
    }
    // ... reconciliation with external API calls ...
}
```

## Упражнение 2: добавление индексирования полей для быстрого поиска

Controller-runtime обеспечивает автоматическое кеширование. Вы можете добавить пользовательские индексы для быстрого поиска.

### Задача 2.1: создайте индексатор полей

Добавьте индексирование в `cmd/main.go` до запуска менеджера:

```go
// In cmd/main.go, after creating manager but before SetupWithManager

// Index databases by environment for fast filtering
if err := mgr.GetFieldIndexer().IndexField(
    context.Background(),
    &databasev1.Database{},
    "spec.environment",
    func(obj client.Object) []string {
        db := obj.(*databasev1.Database)
        if db.Spec.Environment == "" {
            return nil
        }
        return []string{db.Spec.Environment}
    },
); err != nil {
    setupLog.Error(err, "unable to create field index")
    os.Exit(1)
}
```

### Задача 2.2: используйте индексы в контроллере

```go
// In your reconciler, use MatchingFields for indexed queries
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Fast lookup using index
    prodDatabases := &databasev1.DatabaseList{}
    if err := r.List(ctx, prodDatabases, client.MatchingFields{
        "spec.environment": "production",
    }); err != nil {
        return ctrl.Result{}, err
    }
    
    // Use namespace selector for namespace-scoped queries
    nsDatabases := &databasev1.DatabaseList{}
    if err := r.List(ctx, nsDatabases, client.InNamespace(req.Namespace)); err != nil {
        return ctrl.Result{}, err
    }
    
    // ... rest of reconciliation
}
```

## Упражнение 3: оптимизация согласования

### Задача 3.1: пакетные операции

```go
func (r *DatabaseReconciler) reconcileBatch(ctx context.Context, databases []databasev1.Database) error {
    // Group by operation
    var toCreate, toUpdate []databasev1.Database
    
    for _, db := range databases {
        if db.Status.Phase == "" {
            toCreate = append(toCreate, db)
        } else {
            toUpdate = append(toUpdate, db)
        }
    }
    
    // Batch create
    for _, db := range toCreate {
        if err := r.reconcileDatabase(ctx, &db); err != nil {
            return err
        }
    }
    
    // Batch update
    for _, db := range toUpdate {
        if err := r.reconcileDatabase(ctx, &db); err != nil {
            return err
        }
    }
    
    return nil
}
```

## Упражнение 4: мониторинг производительности со встроенными метриками

Controller-runtime автоматически экспонирует метрики. У вашего postgres-operator уже настроены пользовательские метрики!

### Задача 4.1: изучите существующий код метрик

У вашего оператора уже есть метрики в `internal/controller/metrics.go`:

```bash
cd ~/postgres-operator

# Review the metrics file
cat internal/controller/metrics.go
```

Вы должны увидеть уже определённые пользовательские метрики:

```go
var (
    // ReconcileTotal counts the total number of reconciliations
    ReconcileTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "database_reconcile_total",
            Help: "Total number of reconciliations per controller",
        },
        []string{"result"}, // success, error, requeue
    )

    // ReconcileDuration measures the duration of reconciliations
    ReconcileDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "database_reconcile_duration_seconds",
            Help:    "Duration of reconciliations in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"result"},
    )

    // DatabasesTotal tracks the current number of Database resources
    DatabasesTotal = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "database_resources_total",
            Help: "Current number of Database resources by phase",
        },
        []string{"phase"},
    )

    // DatabaseInfo provides information about each database
    DatabaseInfo = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "database_info",
            Help: "Information about Database resources",
        },
        []string{"name", "namespace", "image", "phase"},
    )
)

func init() {
    // Register custom metrics with the global registry
    metrics.Registry.MustRegister(
        ReconcileTotal,
        ReconcileDuration,
        DatabasesTotal,
        DatabaseInfo,
    )
}
```

### Задача 4.2: изучите использование метрик в контроллере

Проверьте, как метрики используются в функции Reconcile:

```bash
# See how metrics are recorded in the reconcile loop
grep -A 10 "Defer metrics" internal/controller/database_controller.go
```

Вы должны увидеть:

```go
// Defer metrics recording
defer func() {
    duration := time.Since(start).Seconds()
    ReconcileDuration.WithLabelValues(reconcileResult).Observe(duration)
    ReconcileTotal.WithLabelValues(reconcileResult).Inc()
}()
```

И установку метрик информации о базе данных:

```go
DatabaseInfo.WithLabelValues(
    db.Name,
    db.Namespace,
    db.Spec.Image,
    db.Status.Phase,
).Set(1)
```

### Задача 4.3: доступ к эндпоинту метрик

Эндпоинт метрик требует аутентификации с bearer-токеном. Мы используем токен ServiceAccount для аутентификации.

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

# Restart operator if already deployed
kubectl rollout restart deploy -n postgres-operator-system postgres-operator-controller-manager

# Port forward to metrics endpoint (using HTTPS on port 8443)
kubectl port-forward -n postgres-operator-system \
  svc/postgres-operator-controller-manager-metrics-service 8443:8443 &

# Get a token for authentication (use the controller-manager service account)
TOKEN=$(kubectl create token postgres-operator-controller-manager -n postgres-operator-system)

# create database to generate some metrics
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: valid-db
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# View custom database metrics (using -k for self-signed cert, -H for auth header)
curl -sk -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics | grep database_

# View reconciliation metrics
curl -sk -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics | grep database_reconcile

# View controller-runtime built-in metrics
curl -sk -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics | grep controller_runtime

# View all metrics
curl -sk -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics | head -100

# Stop port-forward
pkill -f "port-forward.*8443"
```

**Примечание:** эндпоинт метрик использует RBAC Kubernetes для авторизации. У ServiceAccount должна быть ClusterRole `metrics-reader` (настроена в Лабораторной 7.2).

### Задача 4.4: просмотр метрик в Prometheus

Если у вас настроен Prometheus (из Лабораторной 7.2), просмотрите метрики там:

```bash
# Port forward to Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Open http://localhost:9090 and query:
# - database_reconcile_total
# - database_reconcile_duration_seconds
# - database_resources_total
# - database_info

# Stop port-forward when done
pkill -f "port-forward.*9090"
```

**Примеры запросов Prometheus:**

| Запрос | Описание |
|-------|-------------|
| `database_reconcile_total` | Всего согласований по результату |
| `rate(database_reconcile_total[5m])` | Согласований в секунду |
| `database_reconcile_duration_seconds_bucket` | Гистограмма задержки согласования |
| `histogram_quantile(0.99, rate(database_reconcile_duration_seconds_bucket[5m]))` | Задержка p99 |
| `database_resources_total` | Текущие базы данных по фазе |
| `database_info` | Информация о каждой базе данных |

## Упражнение 5: нагрузочное тестирование

### Задача 5.1: создайте много ресурсов

```bash
# Create multiple databases for load testing
for i in {1..50}; do
  kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-db-$i
  namespace: default
spec:
  image: postgres:14
  replicas: 1
  databaseName: db$i
  username: admin
  storage:
    size: 1Gi
EOF
done

echo "Created 50 test databases"
```

### Задача 5.2: отслеживайте производительность под нагрузкой

**Примечание:** `kubectl top` требует установленного metrics-server. Скрипт настройки курса (`scripts/setup-kind-cluster.sh`) устанавливает его автоматически. Если вы получаете «Metrics API not available», установите его вручную:

```bash
# Install metrics-server (if not already installed)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for kind (disable TLS verification for kubelet)
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"},
  {"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-preferred-address-types=InternalIP"}
]'

# Wait for it to be ready
kubectl rollout status deployment/metrics-server -n kube-system
```

Теперь отслеживайте производительность:

```bash
# Watch operator resource usage (requires metrics-server)
watch kubectl top pods -n postgres-operator-system -l control-plane=controller-manager

# Alternative: Check resource requests/limits if metrics-server not available
kubectl get pods -n postgres-operator-system -l control-plane=controller-manager -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'

# In another terminal, watch reconciliation metrics
# Port forward to metrics endpoint (using HTTPS on port 8443)
kubectl port-forward -n postgres-operator-system \
  svc/postgres-operator-controller-manager-metrics-service 8443:8443 &

# Get a token for authentication (use the controller-manager service account)
TOKEN=$(kubectl create token postgres-operator-controller-manager -n postgres-operator-system)

while true; do
  curl -sk -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics 2>/dev/null | grep database_reconcile_total
  sleep 5
done

# Check queue length
curl -sk -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics | grep workqueue

# Check controller logs for reconciliation activity
kubectl logs -n postgres-operator-system -l control-plane=controller-manager --tail=20 -f
```

### Задача 5.3: убедитесь, что все ресурсы согласованы

```bash
# Check status of all databases
kubectl get databases -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,READY:.status.ready

# Count databases in each phase
kubectl get databases -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}' | sort | uniq -c
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
- Настроили ограничение частоты controller-runtime
- Добавили индексирование полей для быстрого поиска
- Оптимизировали согласование с помощью MaxConcurrentReconciles
- Добавили пользовательские метрики производительности
- Провели нагрузочное тестирование оператора с множеством ресурсов

## Ключевые уроки

1. У Controller-runtime есть встроенное ограничение частоты через опцию RateLimiter
2. MaxConcurrentReconciles контролирует параллелизм
3. Индексы полей обеспечивают быстрые фильтрованные запросы
4. Встроенные метрики доступны на `:8443/metrics`
5. Пользовательские метрики используют клиент prometheus с metrics.Registry
6. Нагрузочное тестирование подтверждает производительность оператора при масштабе
7. `client.MatchingFields{}` задействует индексы для быстрого поиска

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Rate Limiter](../solutions/ratelimiter.go) — полная реализация ограничения частоты
- [Performance Optimizations](../solutions/performance.go) — пакетная обработка, кеширование, метрики

## Поздравляем!

Вы завершили Модуль 7! Теперь вы понимаете:
- Упаковку и распространение
- RBAC и безопасность
- Высокую доступность
- Оптимизацию производительности

В Модуле 8 вы изучите продвинутые темы и практические паттерны!

**Навигация:** [← Предыдущая лабораторная: HA](lab-03-high-availability.md) | [Связанный урок](../lessons/04-performance-scalability.md) | [Обзор модуля](../README.md)
