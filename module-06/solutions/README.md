# Решения Модуля 6

Этот каталог содержит полные рабочие решения для лабораторных Модуля 6.

## Файлы

### Тестирование (Лабораторные 1–3)
- [**suite_test.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/suite_test.go): полная настройка набора тестов с envtest
- [**database_controller_test.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/database_controller_test.go): полные примеры модульных тестов
- [**integration_test.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/integration_test.go): полные примеры интеграционных тестов

### Наблюдаемость (Лабораторная 4)
- [**metrics.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/metrics.go): пользовательские метрики Prometheus
- [**observability.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/observability.go): паттерны структурированного логирования и генерации событий
- [**metrics_reader_role_binding.yaml**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/metrics_reader_role_binding.yaml): RBAC-привязка для доступа к метрикам
- [**rbac_kustomization.yaml**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-06/solutions/rbac_kustomization.yaml): обновлённый kustomization, включающий привязку метрик

## Использование

Эти решения можно использовать как:
- Справочный материал при написании собственных тестов
- Отправную точку, если вы застряли
- Примеры лучших практик тестирования

## Интеграция

Чтобы использовать эти решения в вашем операторе:

### 1. Для модульных тестов (envtest)

Скопируйте файлы набора и тестов в каталог контроллера:

```bash
# Copy suite_test.go to internal/controller/suite_test.go
cp suite_test.go ~/postgres-operator/internal/controller/suite_test.go

# Copy test examples to internal/controller/database_controller_test.go
cp database_controller_test.go ~/postgres-operator/internal/controller/database_controller_test.go

# Run tests
cd ~/postgres-operator
make test
# Or: ginkgo -v ./internal/controller/...
```

### 2. Для интеграционных тестов (реальный кластер)

Интеграционные тесты требуют:
- Запущенного кластера Kubernetes (kind, minikube и т. д.)
- Развёрнутого в кластере оператора
- **Типы CRD, зарегистрированные в схеме** (критически важно!)

```bash
# Create integration test directory
mkdir -p ~/postgres-operator/test/integration

# Copy integration_test.go (contains both suite setup and tests)
# Split into two files for your project:

# 1. Create integration_suite_test.go with BeforeSuite (scheme registration)
# 2. Create database_test.go with the Describe blocks

# Ensure operator is deployed
make deploy IMG=<your-image>

# Run integration tests
ginkgo -v ./test/integration

# Skip webhook tests if webhooks aren't deployed
ginkgo -v -skip="webhook" ./test/integration
```

### 3. Для наблюдаемости

```bash
# Step 1: Add RBAC for metrics access
cp metrics_reader_role_binding.yaml ~/postgres-operator/config/rbac/metrics_reader_role_binding.yaml

# Step 2: Update config/rbac/kustomization.yaml to include the new file
# Add this line after 'metrics_reader_role.yaml':
#   - metrics_reader_role_binding.yaml
# (See rbac_kustomization.yaml for the complete file)

# Step 3: Copy metrics code to internal/controller/metrics.go
cp metrics.go ~/postgres-operator/internal/controller/metrics.go

# Step 4: Add event recorder to your controller struct (see observability.go)
# Step 5: Update Reconcile function with metrics and events

# Step 6: Redeploy the operator
cd ~/postgres-operator
make deploy IMG=<your-image>
```

## Ключевые моменты

### Регистрация в схеме (интеграционные тесты)

Интеграционные тесты **обязаны** регистрировать пользовательские типы в схеме:

```go
// In BeforeSuite
err := databasev1.AddToScheme(scheme.Scheme)
Expect(err).NotTo(HaveOccurred())

// Pass scheme to client
k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
```

Без этого вы получите: `no kind is registered for the type v1.Database`

### Используйте k8sClient.Scheme() в модульных тестах

При создании реконсайлера в модульных тестах используйте:

```go
reconciler := &DatabaseReconciler{
    Client: k8sClient,
    Scheme: k8sClient.Scheme(),  // NOT scheme.Scheme
}
```

### Помощники указателей

Используйте `k8s.io/utils/ptr` для помощников указателей:

```go
import "k8s.io/utils/ptr"

Replicas: ptr.To(int32(1))
```

## Команды тестирования

```bash
# Run unit tests (envtest)
cd ~/postgres-operator
make test

# Run unit tests with Ginkgo directly
ginkgo -v ./internal/controller/...

# Run integration tests (requires deployed operator)
ginkgo -v ./test/integration

# Skip webhook tests
ginkgo -v -skip="webhook" ./test/integration

# Run with coverage
go test -coverprofile=coverage.out ./internal/controller/...
go tool cover -html=coverage.out -o coverage.html

# Check metrics (after deploying operator)
kubectl port-forward -n postgres-operator-system deployment/postgres-operator-controller-manager 8080:8080
curl http://localhost:8080/metrics | grep database_
```

## Примечания

- Это полные рабочие примеры
- Они следуют лучшим практикам из уроков
- Тесты используют Ginkgo/Gomega для структуры в стиле BDD
- Модульные тесты используют envtest для легковесного API Kubernetes
- Интеграционные тесты выполняются на реальных кластерах
- Метрики используют клиентскую библиотеку Prometheus
- Логирование использует структурированное логирование (zap)
- События используют регистратор событий Kubernetes
