---
layout: default
title: "Lab 07.2: Rbac Security"
nav_order: 12
parent: "Модуль 7: Подготовка к продакшену"
grand_parent: Модули
mermaid: true
---

# Лабораторная 7.2: Настройка RBAC

**Связанный урок:** [Урок 7.2: RBAC и безопасность](../lessons/02-rbac-security.md)  
**Навигация:** [← Предыдущая лабораторная: Упаковка](lab-01-packaging-distribution.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: HA →](lab-03-high-availability.md)

## Цели

- Проверить и оптимизировать разрешения RBAC
- Настроить service accounts
- Применить лучшие практики безопасности
- Просканировать образы на уязвимости
- Настроить сетевые политики для изоляции оператора

## Предварительные требования

- Завершение [Лабораторной 7.1](lab-01-packaging-distribution.md)
- Оператор со сгенерированным RBAC
- Понимание концепций RBAC
- Кластер kind, созданный с помощью `scripts/setup-kind-cluster.sh` (включает CNI Calico и Prometheus)

## Упражнение 1: проверка сгенерированного RBAC

Kubebuilder генерирует манифесты RBAC автоматически из маркеров в коде вашего контроллера.

### Задача 1.1: проверьте маркеры RBAC в контроллере

Сначала изучите маркеры RBAC вашего контроллера:

```bash
# Navigate to your operator project
cd ~/postgres-operator

# View RBAC markers in your controller
grep -n "// +kubebuilder:rbac" internal/controller/database_controller.go
```

Вы должны увидеть маркеры вроде:

```go
// +kubebuilder:rbac:groups=database.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.example.com,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create;update;patch;delete
```

### Задача 1.2: сгенерируйте и проверьте манифесты RBAC

```bash
# Generate RBAC manifests from markers
make manifests

# View generated ClusterRole
cat config/rbac/role.yaml

# View generated ClusterRoleBinding
cat config/rbac/role_binding.yaml

# View ServiceAccount
cat config/rbac/service_account.yaml
```

### Задача 1.3: проверьте наличие слишком широких разрешений

```bash
# Look for wildcards that might indicate too broad permissions
grep -E "(verbs: \[\"\*\"\]|resources: \[\"\*\"\]|apiGroups: \[\"\*\"\])" config/rbac/role.yaml

# If any are found, review and restrict the corresponding markers
```

## Упражнение 2: оптимизация RBAC

### Задача 2.1: проаудируйте необходимые разрешения

Проверьте, к каким ресурсам ваш контроллер действительно обращается:

```bash
# Find all r.Get, r.Create, r.Update, r.Delete, r.List calls
grep -E "r\.(Get|Create|Update|Delete|List|Patch)" internal/controller/database_controller.go
```

### Задача 2.2: обновите маркеры RBAC

Отредактируйте контроллер так, чтобы он соответствовал только реально нужным разрешениям:

```go
// internal/controller/database_controller.go

// Only include markers for resources you actually use:
// +kubebuilder:rbac:groups=database.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.example.com,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create

// Note: Removed 'delete' from secrets if your controller doesn't delete secrets
// Note: Consider if you need 'update;patch' for all resources
```

### Задача 2.3: перегенерируйте RBAC

```bash
# Regenerate with optimized markers
make manifests

# Review the updated RBAC
cat config/rbac/role.yaml

# Compare rules - they should be minimal
```

## Упражнение 3: проверка конфигурации безопасности Kubebuilder

Kubebuilder генерирует конфигурацию безопасности по умолчанию. Проверим и улучшим её.

### Задача 3.1: проверьте сгенерированный ServiceAccount

```bash
# Kubebuilder creates ServiceAccount automatically
cat config/rbac/service_account.yaml
```

### Задача 3.2: проверьте контекст безопасности развёртывания

Сгенерированное развёртывание kubebuilder включает контексты безопасности. Проверьте их:

```bash
cat config/manager/manager.yaml
```

Найдите настройки безопасности:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
      - name: manager
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
```

### Задача 3.3: усильте контекст безопасности (опционально)

Добавьте дополнительное усиление безопасности в `config/manager/manager.yaml`:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        fsGroup: 65532
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: manager
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

## Упражнение 4: сканирование безопасности

### Задача 4.1: установите Trivy

```bash
# Install Trivy
brew install trivy  # macOS
# or
sudo apt-get install trivy  # Linux
```

### Задача 4.2: просканируйте образ

```bash
# Scan image
trivy image postgres-operator:latest

# Scan with JSON output
trivy image -f json -o scan-report.json postgres-operator:latest

# Fix high/critical vulnerabilities
```

## Упражнение 5: применение усиления безопасности

### Задача 5.1: обновите Dockerfile

```dockerfile
# Use distroless base
FROM gcr.io/distroless/static:nonroot

# Run as non-root
USER 65532:65532
```

## Упражнение 5: включение сетевых политик

Kubebuilder уже генерирует сетевые политики для вашего оператора! Проверим и включим их.

### Задача 5.1: проверьте сгенерированные сетевые политики

Kubebuilder создаёт сетевые политики в `config/network-policy/`:

```bash
cd ~/postgres-operator

# List the generated network policy files
ls -la config/network-policy/

# Review the metrics traffic policy
cat config/network-policy/allow-metrics-traffic.yaml

# Review the webhook traffic policy  
cat config/network-policy/allow-webhook-traffic.yaml
```

**Что генерирует Kubebuilder:**

1. **`allow-metrics-traffic.yaml`** — контролирует доступ к эндпоинту метрик:
   - Разрешает ingress только из пространств имён с меткой `metrics: enabled`
   - Ограничивает портом 8443 (HTTPS-метрики)

2. **`allow-webhook-traffic.yaml`** — контролирует доступ к серверу вебхуков:
   - Разрешает ingress только из пространств имён с меткой `webhook: enabled`
   - Ограничивает портом 443 (HTTPS вебхука)

### Задача 5.2: разберитесь в сетевых политиках

Просмотрите политику метрик:

```yaml
# config/network-policy/allow-metrics-traffic.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metrics-traffic
  namespace: system
spec:
  podSelector:
    matchLabels:
      control-plane: controller-manager
      app.kubernetes.io/name: postgres-operator
  policyTypes:
    - Ingress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            metrics: enabled  # Only from namespaces with this label
      ports:
        - port: 8443
          protocol: TCP
```

**Ключевые моменты:**
- Применяется к подам с меткой `control-plane: controller-manager`
- Разрешает только ingress (входящий трафик)
- Требует, чтобы у пространства имён-источника была метка `metrics: enabled`
- Это означает, что Prometheus должен работать в помеченном пространстве имён, чтобы собирать метрики

### Задача 5.3: включите сетевые политики в Kustomization

Сетевые политики по умолчанию закомментированы. Включите их:

```bash
# View the current kustomization
cat config/default/kustomization.yaml | grep -A 5 "NETWORK POLICY"
```

Вы увидите:
```yaml
# [NETWORK POLICY] Protect the /metrics endpoint and Webhook Server with NetworkPolicy.
#- ../network-policy
```

Отредактируйте `config/default/kustomization.yaml` и раскомментируйте строку network-policy:

```yaml
# [NETWORK POLICY] Protect the /metrics endpoint and Webhook Server with NetworkPolicy.
- ../network-policy
```

### Задача 5.4: разверните с сетевыми политиками

```bash
# For Docker: Deploy the operator with network policies enabled
kind load docker-image postgres-operator:latest --name k8s-operators-course
make deploy IMG=postgres-operator:latest

# For Podman: Deploy operator - use localhost/ prefix to match the loaded image
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar
make deploy IMG=localhost/postgres-operator:latest

# Verify network policies were created
kubectl get networkpolicy -n postgres-operator-system

# View the network policies
kubectl describe networkpolicy -n postgres-operator-system
```

Ожидаемый вывод:
```
NAME                    POD-SELECTOR                                                    AGE
allow-metrics-traffic   app.kubernetes.io/name=postgres-operator,control-plane=...     10s
allow-webhook-traffic   app.kubernetes.io/name=postgres-operator,control-plane=...     10s
```

### Задача 5.5: пометьте пространства имён метками для доступа

Чтобы Prometheus мог собирать метрики, а вебхуки работали, пометьте соответствующие пространства имён.

**Примечание:** если вы использовали скрипт настройки курса (`scripts/setup-kind-cluster.sh`), пространство имён `monitoring` уже помечено меткой `metrics=enabled`.

```bash
# Check if monitoring namespace already has the label
kubectl get namespace monitoring --show-labels

# If not labeled, add it (the setup script does this automatically)
kubectl label namespace monitoring metrics=enabled --overwrite

# Label namespaces where you'll create Database CRs (for webhook access)
kubectl label namespace default webhook=enabled

# Verify labels
kubectl get namespaces --show-labels | grep -E "(metrics|webhook)"
```

### Задача 5.6: включите ServiceMonitor для Prometheus

Kubebuilder генерирует `ServiceMonitor` в `config/prometheus/`, который указывает Prometheus, как собирать метрики вашего оператора. По умолчанию он отключён.

#### Шаг 1: проверьте сгенерированный ServiceMonitor

```bash
# View the kubebuilder-generated ServiceMonitor
cat config/prometheus/monitor.yaml
```

Вы увидите:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: controller-manager-metrics-monitor
spec:
  endpoints:
    - path: /metrics
      port: https
      scheme: https
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        insecureSkipVerify: true
  selector:
    matchLabels:
      control-plane: controller-manager
```

#### Шаг 2: включите ServiceMonitor

Отредактируйте `config/default/kustomization.yaml` и раскомментируйте строку prometheus:

```bash
# Find the PROMETHEUS section
grep -n "PROMETHEUS" config/default/kustomization.yaml
```

Раскомментируйте `- ../prometheus`:
```yaml
# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.
- ../prometheus  # <-- Uncomment this line
```

#### Шаг 3: предоставьте Prometheus доступ RBAC к метрикам

**Важно:** эндпоинт метрик оператора требует аутентификации И авторизации. По умолчанию доступ есть только у ServiceAccount контроллера-менеджера. Нужно предоставить доступ и Prometheus.

Создайте `config/rbac/metrics_reader_prometheus_binding.yaml`:

```yaml
# config/rbac/metrics_reader_prometheus_binding.yaml
# Grant Prometheus ServiceAccount permission to read metrics
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/name: postgres-operator
    app.kubernetes.io/managed-by: kustomize
  name: metrics-reader-prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: metrics-reader
subjects:
- kind: ServiceAccount
  name: prometheus-kube-prometheus-prometheus
  namespace: monitoring
```

Добавьте его в `config/rbac/kustomization.yaml`:

```yaml
resources:
- service_account.yaml
- role.yaml
- role_binding.yaml
- leader_election_role.yaml
- leader_election_role_binding.yaml
- metrics_auth_role.yaml
- metrics_auth_role_binding.yaml
- metrics_reader_role.yaml
- metrics_reader_role_binding.yaml
- metrics_reader_prometheus_binding.yaml  # <-- Add this line
# ... rest of file
```

#### Шаг 4: переразверните с ServiceMonitor и RBAC

```bash
# Regenerate manifests to include the new RBAC binding
make manifests

# For Docker: Redeploy to include the ServiceMonitor
make deploy IMG=postgres-operator:latest

# For Podman: Redeploy to include the ServiceMonitor
make deploy IMG=localhost/postgres-operator:latest

# Verify the ServiceMonitor was created
kubectl get servicemonitor -n postgres-operator-system

# Verify Prometheus RBAC binding was created
kubectl get clusterrolebinding | grep metrics-reader
```

Ожидаемый вывод:
```
NAME                                              AGE
postgres-operator-metrics-reader-prometheus       10s
postgres-operator-metrics-reader-rolebinding      10s
```

**Примечание:** скрипт настройки курса (`scripts/setup-kind-cluster.sh`) настраивает Prometheus на обнаружение ServiceMonitor из всех пространств имён без требования конкретных меток. Если вы используете другую установку Prometheus, возможно, придётся добавить метку `release: prometheus` к вашему ServiceMonitor.

### Задача 5.7: убедитесь, что Prometheus может собирать метрики

Теперь давайте убедимся, что Prometheus собирает метрики вашего оператора.

#### Шаг 1: запустите port-forward к Prometheus

```bash
# Start port-forward in background
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Note the PID for later cleanup
PF_PID=$!
echo "Port-forward PID: $PF_PID"
```

#### Шаг 2: откройте UI Prometheus

Откройте браузер и перейдите по адресу: **http://localhost:9090**

#### Шаг 3: проверьте, собирается ли цель вашего оператора

1. В UI Prometheus нажмите **Status** → **Targets** в верхнем меню
2. Найдите цель с `serviceMonitor/postgres-operator-system/` в имени
3. **State** должен показывать `UP` (зелёный)

Если цель не появляется или показывает `DOWN`, проверьте:
- Развёрнут ли ServiceMonitor? (`kubectl get servicemonitor -n postgres-operator-system`)
- Помечено ли пространство имён monitoring? (`kubectl get ns monitoring --show-labels`)
- Не блокируют ли доступ сетевые политики?
- Настроен ли Prometheus на обнаружение всех ServiceMonitor?

```bash
# Check if Prometheus discovers all ServiceMonitors (should be empty selector)
kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'
# Empty {} means it discovers all ServiceMonitors

# If it shows 'release: prometheus', upgrade Prometheus with the course setup settings
# or add 'release: prometheus' label to your ServiceMonitor
```

#### Шаг 4: запросите метрики оператора

1. Вернитесь на главную страницу Prometheus (нажмите логотип **Prometheus** или **Graph**)
2. В поле ввода **Expression** введите один из этих запросов:
   
   ```promql
   controller_runtime_reconcile_total
   ```
   
3. Нажмите кнопку **Execute** (или Enter)
4. Нажмите вкладку **Graph**, чтобы увидеть визуализацию временны́х рядов

**Распространённые метрики оператора для изучения:**

| Метрика | Описание |
|--------|-------------|
| `controller_runtime_reconcile_total` | Всего согласований по контроллеру и результату |
| `controller_runtime_reconcile_errors_total` | Всего ошибок согласования |
| `controller_runtime_reconcile_time_seconds` | Время, затраченное на согласование |
| `workqueue_depth` | Текущая глубина рабочей очереди |
| `workqueue_adds_total` | Всего элементов, добавленных в очередь |

#### Шаг 5: примеры запросов для пробы

Вставьте их в поле Expression Prometheus:

```promql
# Reconciliation rate per second (last 5 minutes)
rate(controller_runtime_reconcile_total[5m])

# Error rate
rate(controller_runtime_reconcile_errors_total[5m])

# 99th percentile reconciliation latency
histogram_quantile(0.99, rate(controller_runtime_reconcile_time_seconds_bucket[5m]))
```

#### Шаг 6: очистка port-forward

```bash
# Stop the port-forward
pkill -f "port-forward.*9090"
```

### Задача 5.8: протестируйте применение сетевой политики

Скрипт настройки курса устанавливает **Calico CNI**, который применяет сетевые политики. Проверим, что это работает.

#### Шаг 1: убедитесь, что Calico запущен

```bash
# Check Calico pods are running
kubectl get pods -n kube-system | grep calico

# Expected output:
# calico-kube-controllers-xxx   1/1     Running
# calico-node-xxx               1/1     Running
```

#### Шаг 2: протестируйте доступ из непомеченного пространства имён

```bash
# Create a test namespace WITHOUT the metrics=enabled label
kubectl create namespace test-netpol

# Try to access the operator metrics from the unlabeled namespace
# This should FAIL (timeout or connection refused) because of the NetworkPolicy
kubectl run test-curl -n test-netpol --rm -it --image=curlimages/curl --restart=Never -- \
  curl -k --connect-timeout 5 https://postgres-operator-controller-manager-metrics-service.postgres-operator-system:8443/metrics

# Expected: curl: (28) Connection timed out or similar error
```

#### Шаг 3: протестируйте доступ из помеченного пространства имён

```bash
# Label the test namespace to allow metrics access
kubectl label namespace test-netpol metrics=enabled

# Try again - this should SUCCEED
kubectl run test-curl2 -n test-netpol --rm -it --image=curlimages/curl --restart=Never -- \
  curl -k --connect-timeout 5 https://postgres-operator-controller-manager-metrics-service.postgres-operator-system:8443/metrics

# Expected: Metrics output (or 401 Unauthorized if auth is required, but connection succeeds)
```

#### Шаг 4: очистка

```bash
kubectl delete namespace test-netpol
```

**Ключевой вывод:** сетевые политики обеспечивают, что только поды в пространствах имён с меткой `metrics=enabled` могут получить доступ к эндпоинту метрик. Это эшелонированная защита!

## Очистка

```bash
# Undeploy operator (this also removes the network policy)
make undeploy

# If you need to remove network policy separately
kubectl delete networkpolicy controller-manager -n postgres-operator-system

# Uninstall CRDs
make uninstall
```

## Итоги лабораторной

В этой лабораторной вы:
- Проверили маркеры RBAC в контроллерах kubebuilder
- Сгенерировали и проверили манифесты RBAC с помощью `make manifests`
- Оптимизировали разрешения RBAC по принципу наименьших привилегий
- Проверили конфигурации безопасности kubebuilder
- Просканировали образы на уязвимости с помощью Trivy
- Усилили безопасность с помощью контекстов безопасности
- Включили сетевые политики, сгенерированные kubebuilder
- Пометили пространства имён метками для разрешения трафика метрик и вебхуков
- Включили ServiceMonitor для сбора метрик Prometheus
- Проверили сбор метрик в UI Prometheus

## Ключевые уроки

1. RBAC генерируется из маркеров через `make manifests`
2. Проверяйте `config/rbac/role.yaml` на сгенерированные разрешения
3. Минимизируйте маркеры под реальные нужды контроллера
4. Kubebuilder по умолчанию включает контексты безопасности
5. Регулярно сканируйте образы с помощью Trivy или аналогичных инструментов
6. **Kubebuilder генерирует сетевые политики** в `config/network-policy/`
7. Включайте сетевые политики, раскомментировав `../network-policy` в kustomization
8. Помечайте пространства имён метками `metrics: enabled` или `webhook: enabled` для разрешения доступа
9. **Kubebuilder генерирует ServiceMonitor** в `config/prometheus/` — включите его!
10. **Предоставьте Prometheus доступ RBAC** к эндпоинту метрик (шаг 3 выше)
11. Используйте **Status → Targets** в UI Prometheus, чтобы убедиться, что сбор работает
12. Скрипт настройки курса настраивает Prometheus на обнаружение всех ServiceMonitor
13. Базовый образ distroless уже используется kubebuilder
14. Сетевые политики требуют CNI, который их поддерживает (Calico)

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [RBAC Configuration](../solutions/rbac.yaml) — оптимизированный RBAC по принципу наименьших привилегий
- [Security Configuration](../solutions/security.yaml) — контексты безопасности, сетевые политики

## Дальнейшие шаги

Теперь давайте реализуем высокую доступность!

**Навигация:** [← Предыдущая лабораторная: Упаковка](lab-01-packaging-distribution.md) | [Связанный урок](../lessons/02-rbac-security.md) | [Следующая лабораторная: HA →](lab-03-high-availability.md)
