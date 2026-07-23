---
layout: default
title: "Lab 06.4: Debugging Observability"
nav_order: 14
parent: "Модуль 6: Тестирование и отладка"
grand_parent: Модули
mermaid: true
---

# Лабораторная 6.4: Добавление наблюдаемости

**Связанный урок:** [Урок 6.4: Отладка и наблюдаемость](../lessons/04-debugging-observability.md)  
**Навигация:** [← Предыдущая лабораторная: Интеграционное тестирование](lab-03-integration-testing.md) | [Обзор модуля](../README.md)

## Цели

- Разобраться в существующем структурированном логировании
- Добавить пользовательские метрики Prometheus
- Добавить генерацию событий Kubernetes
- Настроить отладку с Delve
- Проверить, что все возможности наблюдаемости работают

## Предварительные требования

- Завершение [Лабораторной 6.3](lab-03-integration-testing.md)
- Оператор Database, развёрнутый в кластере
- Понимание концепций наблюдаемости

## Упражнение 1: проверка структурированного логирования

Kubebuilder уже настраивает структурированное логирование с zap. Проверим, что оно работает.

### Задача 1.1: проверьте существующую конфигурацию логирования

В вашем `cmd/main.go` логирование уже настроено:

```go
opts := zap.Options{
    Development: true,
}
opts.BindFlags(flag.CommandLine)
flag.Parse()

ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))
```

### Задача 1.2: проверьте логирование в контроллере

Ваш контроллер уже использует структурированное логирование. Проверьте `internal/controller/database_controller.go`:

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    // ... later in the code:
    logger.Info("Reconciling Database", "name", db.Name)
    logger.Info("STATE TRANSITION: Pending -> Provisioning", "database", db.Name)
}
```

### Задача 1.3: протестируйте логирование

```bash
# Deploy the operator (if not already deployed)
cd ~/postgres-operator
make deploy IMG=postgres-operator:latest

# Watch logs in real-time
kubectl logs -n postgres-operator-system -l control-plane=controller-manager -f

# In another terminal, create a Database to trigger reconciliation
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-logging
  namespace: default
spec:
  image: postgres:14
  databaseName: testdb
  username: testuser
  storage:
    size: 1Gi
EOF
```

**Ожидаемый вывод** (в терминале с логами):
```
INFO    Reconciling Database    {"controller": "database", "name": "test-logging"}
INFO    STATE TRANSITION: Pending -> Provisioning    {"database": "test-logging"}
INFO    Creating Secret    {"name": "test-logging-credentials"}
INFO    Creating StatefulSet    {"name": "test-logging"}
```

### Задача 1.4: очистка

```bash
kubectl delete database test-logging
```

## Упражнение 2: добавление метрик Prometheus

### Задача 2.1: добавьте RBAC-привязку для метрик

Каркас Kubebuilder создаёт ClusterRole `metrics-reader`, но не привязывает его ни к кому. Нужно создать привязку, чтобы ServiceAccount мог обращаться к своим метрикам.

Создайте `config/rbac/metrics_reader_role_binding.yaml`:

```bash
cat > ~/postgres-operator/config/rbac/metrics_reader_role_binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: postgres-operator
    app.kubernetes.io/managed-by: kustomize
  name: metrics-reader-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: metrics-reader
subjects:
- kind: ServiceAccount
  name: controller-manager
  namespace: system
EOF
```

Обновите `config/rbac/kustomization.yaml`, чтобы включить новый файл:

```bash
cat > ~/postgres-operator/config/rbac/kustomization.yaml << 'EOF'
resources:
# All RBAC will be applied under this service account in
# the deployment namespace. You may comment out this resource
# if your manager will use a service account that exists at
# runtime. Be sure to update RoleBinding and ClusterRoleBinding
# subjects if changing service account names.
- service_account.yaml
- role.yaml
- role_binding.yaml
- leader_election_role.yaml
- leader_election_role_binding.yaml
# The following RBAC configurations are used to protect
# the metrics endpoint with authn/authz. These configurations
# ensure that only authorized users and service accounts
# can access the metrics endpoint. Comment the following
# permissions if you want to disable this protection.
# More info: https://book.kubebuilder.io/reference/metrics.html
- metrics_auth_role.yaml
- metrics_auth_role_binding.yaml
- metrics_reader_role.yaml
- metrics_reader_role_binding.yaml
# For each CRD, "Admin", "Editor" and "Viewer" roles are scaffolded by
# default, aiding admins in cluster management. Those roles are
# not used by the postgres-operator itself. You can comment the following lines
# if you do not want those helpers be installed with your Project.
- database_admin_role.yaml
- database_editor_role.yaml
- database_viewer_role.yaml
EOF
```

### Задача 2.2: создайте файл метрик

Создайте `internal/controller/metrics.go`:

```bash
cat > ~/postgres-operator/internal/controller/metrics.go << 'EOF'
/*
Copyright 2025.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controller

import (
	"github.com/prometheus/client_golang/prometheus"
	"sigs.k8s.io/controller-runtime/pkg/metrics"
)

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
EOF
```

### Задача 2.3: обновите контроллер для использования метрик

Добавьте инструментирование метриками в вашу функцию `Reconcile`. Обновите `internal/controller/database_controller.go`:

**Добавьте импорт:**
```go
import (
    // ... existing imports ...
    "time"
)
```

**Обновите функцию Reconcile** — добавьте в самом начале:

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    start := time.Now()
    reconcileResult := "success"
    
    // Defer metrics recording
    defer func() {
        duration := time.Since(start).Seconds()
        ReconcileDuration.WithLabelValues(reconcileResult).Observe(duration)
        ReconcileTotal.WithLabelValues(reconcileResult).Inc()
    }()
    
    logger := log.FromContext(ctx)
    
    // ... rest of existing code ...
```

**Обновите обработку ошибок** — при возврате ошибок задавайте результат:

```go
    // Example: in error returns, set reconcileResult before returning
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        reconcileResult = "error"  // Add this line
        return ctrl.Result{}, err
    }
```

**Добавьте метрику информации о базе данных** — в функции reconcile после получения базы данных:

```go
    // Record database info metric
    DatabaseInfo.WithLabelValues(
        db.Name,
        db.Namespace, 
        db.Spec.Image,
        db.Status.Phase,
    ).Set(1)
```

### Задача 2.4: пересоберите и разверните

```bash
cd ~/postgres-operator

# Rebuild the operator (for docker)
make docker-build IMG=postgres-operator:latest

# Build with podman (note: image will be localhost/postgres-operator:latest)
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman

# Load into kind (for docker)
kind load docker-image postgres-operator:latest

# For podman: Load image into kind (save to tarball, then load)
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar

# For docker: Deploy operator
make deploy IMG=postgres-operator:latest

# For podman: Deploy operator - use localhost/ prefix to match the loaded image
make deploy IMG=localhost/postgres-operator:latest

# Redeploy
kubectl rollout restart deployment -n postgres-operator-system postgres-operator-controller-manager

# Wait for it to be ready
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n postgres-operator-system --timeout=60s
```

### Задача 2.5: протестируйте метрики

Эндпоинт метрик по умолчанию использует HTTPS с аутентификацией.

**Примечание**: RBAC оператора включает `metrics-reader-rolebinding`, который предоставляет ServiceAccount контроллера разрешение на чтение метрик. Это было добавлено в `config/rbac/metrics_reader_role_binding.yaml`.

```bash
# Create a test database first
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-metrics
  namespace: default
spec:
  image: postgres:14
  databaseName: metricsdb
  username: metricsuser
  storage:
    size: 1Gi
EOF

# Wait for it to be reconciled
sleep 30

# Get a token for the ServiceAccount
TOKEN=$(kubectl create token -n postgres-operator-system postgres-operator-controller-manager)

# Port forward to the metrics service
kubectl port-forward -n postgres-operator-system svc/postgres-operator-controller-manager-metrics-service 8443:8443 &
sleep 2

# Query metrics with the token
curl -k -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics 2>/dev/null | grep database_

# Stop port-forward
pkill -f "port-forward.*8443"
```

**Альтернатива: отключить защищённые метрики для локальной разработки**

Если вы предпочитаете более простой доступ во время разработки:

```bash
# Patch the deployment to disable secure metrics
kubectl patch deployment -n postgres-operator-system postgres-operator-controller-manager \
  --type='json' -p='[
    {"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
      "--metrics-bind-address=:8080",
      "--leader-elect",
      "--health-probe-bind-address=:8081",
      "--metrics-secure=false"
    ]}
  ]'

# Wait for rollout
kubectl rollout status deployment -n postgres-operator-system postgres-operator-controller-manager

# Port forward and query metrics (no auth needed)
kubectl port-forward -n postgres-operator-system deployment/postgres-operator-controller-manager 8080:8080 &
sleep 2
curl http://localhost:8080/metrics 2>/dev/null | grep database_

# Stop port-forward
pkill -f "port-forward.*8080"
```

**Ожидаемый вывод:**
```
# HELP database_reconcile_total Total number of reconciliations per controller
# TYPE database_reconcile_total counter
database_reconcile_total{result="success"} 5
# HELP database_reconcile_duration_seconds Duration of reconciliations in seconds
# TYPE database_reconcile_duration_seconds histogram
database_reconcile_duration_seconds_bucket{result="success",le="0.005"} 2
...
# HELP database_info Information about Database resources
# TYPE database_info gauge
database_info{image="postgres:14",name="test-metrics",namespace="default",phase="Provisioning"} 1
```

### Задача 2.6: очистка

```bash
# Stop port-forward
pkill -f "port-forward.*8443"

# Delete test database
kubectl delete database test-metrics
```

## Упражнение 3: добавление событий Kubernetes

### Задача 3.1: добавьте разрешение RBAC для событий

Контроллеру нужно разрешение на создание событий. Добавьте маркер RBAC kubebuilder в `internal/controller/database_controller.go`.

Найдите существующие маркеры RBAC (в начале файла, перед функцией `Reconcile`) и добавьте разрешение на события:

```go
// +kubebuilder:rbac:groups=database.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.example.com,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch  // <-- ADD THIS LINE
```

Затем перегенерируйте манифесты RBAC:

```bash
cd ~/postgres-operator
make manifests
```

Это обновит `config/rbac/role.yaml`, добавив разрешение на события.

### Задача 3.2: добавьте регистратор событий (Event Recorder) в контроллер

Обновите `internal/controller/database_controller.go`:

**Добавьте импорт:**
```go
import (
    // ... existing imports ...
    "k8s.io/client-go/tools/record"
)
```

**Обновите структуру:**
```go
// DatabaseReconciler reconciles a Database object
type DatabaseReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}
```

### Задача 3.3: обновите main.go, чтобы предоставить регистратор событий

Обновите `cmd/main.go`:

```go
if err := (&controller.DatabaseReconciler{
    Client:   mgr.GetClient(),
    Scheme:   mgr.GetScheme(),
    Recorder: mgr.GetEventRecorderFor("database-controller"),
}).SetupWithManager(mgr); err != nil {
```

### Задача 3.4: генерируйте события в контроллере

Добавьте события в ключевых точках вашего контроллера. Обновите `internal/controller/database_controller.go`:

**В `handleProvisioning` после создания StatefulSet:**
```go
func (r *DatabaseReconciler) handleProvisioning(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // ... existing code ...
    
    if errors.IsNotFound(err) {
        logger.Info("Creating StatefulSet", "database", db.Name)
        if err := r.reconcileStatefulSet(ctx, db); err != nil {
            r.Recorder.Event(db, "Warning", "CreateFailed", "Failed to create StatefulSet: "+err.Error())
            return ctrl.Result{}, err
        }
        r.Recorder.Event(db, "Normal", "Created", "StatefulSet created successfully")
        return ctrl.Result{Requeue: true}, nil
    }
    
    // ... rest of existing code ...
}
```

**В `handleVerifying`, когда база данных становится готова:**
```go
func (r *DatabaseReconciler) handleVerifying(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // ... existing code ...
    
    logger.Info("Database is now READY!", "database", db.Name, "endpoint", db.Status.Endpoint)
    r.Recorder.Event(db, "Normal", "Ready", "Database is ready at "+db.Status.Endpoint)
    
    return ctrl.Result{}, r.Status().Update(ctx, db)
}
```

**В `handleDeletion`:**
```go
func (r *DatabaseReconciler) handleDeletion(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // ... at the beginning ...
    r.Recorder.Event(db, "Normal", "Deleting", "Starting cleanup of database resources")
    
    // ... at the end before removing finalizer ...
    r.Recorder.Event(db, "Normal", "Deleted", "Cleanup completed successfully")
    
    // ... rest of code ...
}
```

### Задача 3.5: обновите тестовые файлы (важно!)

Поскольку мы добавили `Recorder` в структуру, обновите `internal/controller/database_controller_test.go`:

```go
// In each test where you create DatabaseReconciler, add the Recorder field:
controllerReconciler := &DatabaseReconciler{
    Client:   k8sClient,
    Scheme:   k8sClient.Scheme(),
    Recorder: record.NewFakeRecorder(100),  // Add this line
}
```

**Добавьте импорт:**
```go
import (
    // ... existing imports ...
    "k8s.io/client-go/tools/record"
)
```

### Задача 3.6: пересоберите и разверните

```
cd ~/postgres-operator

# Run tests first to make sure they pass
make test

# Rebuild the operator (for docker)
make docker-build IMG=postgres-operator:latest

# Build with podman (note: image will be localhost/postgres-operator:latest)
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman

# Load into kind (for docker)
kind load docker-image postgres-operator:latest

# For podman: Load image into kind (save to tarball, then load)
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar

# For docker: Deploy operator
make deploy IMG=postgres-operator:latest

# For podman: Deploy operator - use localhost/ prefix to match the loaded image
make deploy IMG=localhost/postgres-operator:latest

# Redeploy
kubectl rollout restart deployment -n postgres-operator-system postgres-operator-controller-manager

# Wait for it to be ready
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n postgres-operator-system --timeout=60s
```

### Задача 3.7: протестируйте события

```bash
# Create a test database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-events
  namespace: default
spec:
  image: postgres:14
  databaseName: eventsdb
  username: eventsuser
  storage:
    size: 1Gi
EOF

# Wait for reconciliation
sleep 30

# View events for the database
kubectl get events --field-selector involvedObject.name=test-events --sort-by='.lastTimestamp'

# Or view all recent events
kubectl get events -n default --sort-by='.lastTimestamp' | head -20
```

**Ожидаемый вывод:**
```
LAST SEEN   TYPE     REASON    OBJECT                  MESSAGE
30s         Normal   Created   database/test-events    StatefulSet created successfully
15s         Normal   Ready     database/test-events    Database is ready at test-events.default.svc.cluster.local:5432
```

### Задача 3.8: очистка

```bash
kubectl delete database test-events
```

## Упражнение 4: настройка отладчика Delve

**Примечание**: для этого упражнения вы запустите оператор локально (вне кластера) для отладки. Сначала уменьшите масштаб развёрнутого оператора, чтобы он не конфликтовал.

### Задача 4.1: установите Delve

```bash
go install github.com/go-delve/delve/cmd/dlv@latest

# Verify installation
dlv version
```

### Задача 4.2: подготовьтесь к локальной отладке

```bash
cd ~/postgres-operator

# Undeploy the operator (removes deployment, webhooks, and RBAC)
make undeploy

# Verify it's gone
kubectl get pods -n postgres-operator-system

# Reinstall CRDs (undeploy removes them)
make install
```

### Задача 4.3: отладка с Delve

При локальном запуске оператора для отладки:
- **Вебхуки должны быть отключены** — им нужны TLS-сертификаты, которых локально нет
- **CRD должны быть установлены** — уже сделано в предыдущих упражнениях
- **Kubeconfig должен быть валиден** — используется ваш локальный контекст kubectl

```bash
cd ~/postgres-operator

# IMPORTANT: Disable webhooks (they require TLS certs that don't exist locally)
export ENABLE_WEBHOOKS=false

# Start the operator with Delve
dlv debug ./cmd/main.go -- \
  --metrics-bind-address=:8080 \
  --health-probe-bind-address=:8081 \
  --metrics-secure=false
```

Теперь в консоли Delve:

```
# Set a breakpoint in the Reconcile function
(dlv) break internal/controller/database_controller.go:81
Breakpoint 1 set at ...

# Start the operator
(dlv) continue
```

Оператор теперь запущен. В **другом терминале** создайте Database, чтобы запустить согласование:

```bash
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: debug-test
  namespace: default
spec:
  image: postgres:14
  databaseName: debugdb
  username: debuguser
  storage:
    size: 1Gi
EOF
```

Вернувшись в Delve, точка останова должна сработать:

```
# Inspect variables
(dlv) print req
(dlv) print req.NamespacedName

# Step through code
(dlv) next
(dlv) next

# After the db variable is populated:
(dlv) print db.Name
(dlv) print db.Spec

# Continue execution
(dlv) continue

# Exit when done
(dlv) quit
```

### Задача 4.4: очистка сессии отладки

```bash
cd ~/postgres-operator

# For docker: Deploy operator
make deploy IMG=postgres-operator:latest

# For podman: Deploy operator - use localhost/ prefix to match the loaded image
make deploy IMG=localhost/postgres-operator:latest

# Wait for it to be ready
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n postgres-operator-system --timeout=60s

# Delete the test database
kubectl delete database debug-test
```

### Задача 4.5: отладка в VS Code (альтернатива)

Для более удобной отладки создайте `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Operator",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/cmd/main.go",
            "args": [
                "--metrics-bind-address=:8080",
                "--health-probe-bind-address=:8081",
                "--metrics-secure=false"
            ],
            "env": {
                "ENABLE_WEBHOOKS": "false"
            }
        }
    ]
}
```

Затем:
1. Сначала уменьшите масштаб развёрнутого оператора
2. Установите точки останова, кликнув на полосе слева
3. Нажмите F5, чтобы начать отладку
4. В терминале создайте Database, чтобы сработала точка останова
5. VS Code остановится на вашей точке останова

### Задача 4.6: полезные команды Delve

```
break <file>:<line>  - Set breakpoint
continue (c)         - Continue execution  
next (n)             - Step over (next line)
step (s)             - Step into function
print (p) <var>      - Print variable value
locals               - Show all local variables
stack                - Show call stack
goroutines           - List all goroutines
quit (q)             - Exit debugger
```

**Распространённые проблемы:**
- `no such file or directory: tls.crt` → установите `ENABLE_WEBHOOKS=false`
- `Timeout: failed waiting for Informer to sync` → проверьте kubeconfig и связь с кластером
- Точка останова не срабатывает → создайте ресурс Database, чтобы запустить согласование

## Упражнение 5: полная проверка наблюдаемости

### Задача 5.1: разверните и создайте тестовый ресурс

```bash
cd ~/postgres-operator

# For Docker: Make sure latest version is deployed
make docker-build IMG=postgres-operator:latest
kind load docker-image postgres-operator:latest
make deploy IMG=postgres-operator:latest

# For Podman
# Build with podman (note: image will be localhost/postgres-operator:latest)
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
# For podman: Load image into kind (save to tarball, then load)
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar
# For podman: Deploy operator - use localhost/ prefix to match the loaded image
make deploy IMG=localhost/postgres-operator:latest

# Redeploy
kubectl rollout restart deployment -n postgres-operator-system postgres-operator-controller-manager

# Wait for deployment
kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n postgres-operator-system --timeout=120s

# Create test database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: observability-test
  namespace: default
spec:
  image: postgres:14
  databaseName: obsdb
  username: obsuser
  storage:
    size: 1Gi
EOF
```

### Задача 5.2: проверьте все возможности наблюдаемости

```bash
echo "=== 1. Checking Logs ==="
kubectl logs -n postgres-operator-system -l control-plane=controller-manager --tail=50 | grep -E "(Reconciling|STATE TRANSITION|Creating|Ready)"

echo ""
echo "=== 2. Checking Events ==="
kubectl get events --field-selector involvedObject.name=observability-test --sort-by='.lastTimestamp'

echo ""
echo "=== 3. Checking Database Status ==="
kubectl get database observability-test -o jsonpath='{.status}' | jq .

echo ""
echo "=== 4. Checking Metrics ==="
# Note: This assumes you've disabled secure metrics (Option A from Exercise 2)
# If secure metrics are enabled, you'll need to use a token
kubectl port-forward -n postgres-operator-system deployment/postgres-operator-controller-manager 8080:8080 &
sleep 2
curl -s http://localhost:8080/metrics 2>/dev/null | grep -E "^database_" | head -20 || echo "Metrics not available (secure metrics may be enabled)"
pkill -f "port-forward.*8080" 2>/dev/null

# If secure metrics are enabled, use below
# Get a token for the ServiceAccount
TOKEN=$(kubectl create token -n postgres-operator-system postgres-operator-controller-manager)
# Port forward to the metrics service
kubectl port-forward -n postgres-operator-system svc/postgres-operator-controller-manager-metrics-service 8443:8443 &
sleep 2
# Query metrics with the token
curl -k -H "Authorization: Bearer $TOKEN" https://localhost:8443/metrics 2>/dev/null | grep database_
# Stop port-forward
pkill -f "port-forward.*8443"
```

### Задача 5.3: очистка

```bash
kubectl delete database observability-test
```

## Итоги лабораторной

В этой лабораторной вы:
- Проверили, что существующее структурированное логирование работает
- Добавили пользовательские метрики Prometheus для отслеживания согласования
- Добавили генерацию событий Kubernetes для видимости пользователю
- Научились использовать отладчик Delve для устранения неполадок
- Проверили, что все возможности наблюдаемости работают вместе

## Ключевые уроки

1. **Структурированное логирование** — уже настроено Kubebuilder; используйте `log.FromContext(ctx)` с парами «ключ-значение»
2. **Пользовательские метрики** — регистрируйте через `metrics.Registry.MustRegister()` в функции `init()`
3. **Event Recorder** — добавьте в структуру реконсайлера, получайте от менеджера через `mgr.GetEventRecorderFor()`
4. **События видны пользователю** — используйте `Normal` для успеха, `Warning` для ошибок
5. **Обновляйте тесты** — при добавлении полей в структуру реконсайлера обновляйте и тестовые файлы
6. **Отладка Delve** — используйте `ENABLE_WEBHOOKS=false` для локальной отладки
7. **Защищённые метрики** — современный Kubebuilder использует HTTPS с аутентификацией на порту 8443; используйте токен ServiceAccount для доступа
8. **Отключайте защищённые метрики для разработки** — добавьте флаг `--metrics-secure=false` для более простого локального тестирования

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Metrics RBAC Binding](../solutions/metrics_reader_role_binding.yaml) — ClusterRoleBinding для доступа к метрикам
- [RBAC Kustomization](../solutions/rbac_kustomization.yaml) — обновлённый kustomization с привязкой метрик
- [Metrics Implementation](../solutions/metrics.go) — пользовательские метрики Prometheus
- [Observability Examples](../solutions/observability.go) — паттерны логирования и событий

## Поздравляем!

Вы завершили Модуль 6! Теперь вы понимаете:
- Основы и стратегии тестирования
- Модульное тестирование с envtest
- Интеграционное тестирование с реальными кластерами
- Отладку и наблюдаемость

В Модуле 7 вы изучите развёртывание в продакшене и лучшие практики!

**Навигация:** [← Предыдущая лабораторная: Интеграционное тестирование](lab-03-integration-testing.md) | [Связанный урок](../lessons/04-debugging-observability.md) | [Обзор модуля](../README.md)
