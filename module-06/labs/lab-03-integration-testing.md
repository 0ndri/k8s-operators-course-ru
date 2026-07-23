---
layout: default
title: "Lab 06.3: Integration Testing"
nav_order: 13
parent: "Модуль 6: Тестирование и отладка"
grand_parent: Модули
mermaid: true
---

# Лабораторная 6.3: Создание интеграционных тестов

**Связанный урок:** [Урок 6.3: Интеграционное тестирование](../lessons/03-integration-testing.md)  
**Навигация:** [← Предыдущая лабораторная: Модульное тестирование](lab-02-unit-testing-envtest.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Наблюдаемость →](lab-04-debugging-observability.md)

## Цели

- Настроить среду интеграционного тестирования
- Написать сквозные (end-to-end) тесты
- Протестировать полные рабочие процессы
- Интегрировать с CI/CD

## Предварительные требования

- Завершение [Лабораторной 6.2](lab-02-unit-testing-envtest.md)
- Установленный kind
- Понимание интеграционного тестирования

## Упражнение 1: настройка среды интеграционного тестирования

### Задача 1.1: создайте каталог интеграционных тестов

```bash
# Create integration test directory
mkdir -p test/integration
cd test/integration
```

### Задача 1.2: инициализируйте набор Ginkgo

```bash
# Initialize Ginkgo suite
ginkgo bootstrap
```

### Задача 1.3: создайте набор тестов

Создайте `test/integration/integration_suite_test.go`:

**Важно**: клиент должен знать о вашем пользовательском типе `Database`. Вы должны зарегистрировать его в схеме!

```go
package integration_test

import (
	"testing"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	"k8s.io/client-go/kubernetes/scheme"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/client/config"

	databasev1 "github.com/example/postgres-operator/api/v1"
)

var (
	k8sClient client.Client
)

func TestIntegration(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Integration Suite")
}

var _ = BeforeSuite(func() {
	By("setting up integration test environment")

	// Register the Database type with the scheme
	// Without this, the client won't know how to serialize/deserialize Database objects!
	err := databasev1.AddToScheme(scheme.Scheme)
	Expect(err).NotTo(HaveOccurred())

	cfg, err := config.GetConfig()
	Expect(err).NotTo(HaveOccurred())

	// Pass the scheme to the client so it knows about our custom types
	k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
	Expect(err).NotTo(HaveOccurred())
	Expect(k8sClient).NotTo(BeNil())
})
```

**Почему нужна регистрация в схеме?**
- Клиент Kubernetes использует схему для преобразования Go-типов в/из JSON/YAML
- Встроенные типы (Pod, Service и т. д.) уже зарегистрированы
- Типы пользовательских ресурсов, такие как `Database`, нужно регистрировать явно

## Упражнение 2: написание сквозного теста

### Задача 2.1: протестируйте жизненный цикл Database

Создайте `test/integration/database_test.go`:

**Примечание**: пакет должен совпадать с файлом набора (`integration_test`).

```go
package integration_test

import (
    "context"
	"fmt"
    "time"
    
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
	"k8s.io/utils/ptr"
    "sigs.k8s.io/controller-runtime/pkg/client"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
)

var _ = Describe("Database Operator Integration", func() {
    var (
		ctx      context.Context
		cancel   context.CancelFunc
		timeout  = 5 * time.Minute
		interval = 2 * time.Second
    )
    
    BeforeEach(func() {
        ctx, cancel = context.WithCancel(context.Background())
    })
    
    AfterEach(func() {
        cancel()
    })
    
    Context("Database lifecycle", func() {
		var (
			dbName string
			key    types.NamespacedName
		)

		BeforeEach(func() {
			// Use unique name per test to avoid conflicts
			dbName = fmt.Sprintf("integration-test-%d", time.Now().UnixNano())
			key = types.NamespacedName{
				Name:      dbName,
				Namespace: "default",
			}
		})

		AfterEach(func() {
			// Cleanup: delete the Database if it exists
			db := &databasev1.Database{}
			if err := k8sClient.Get(ctx, key, db); err == nil {
				// Remove finalizer to allow deletion
				db.Finalizers = nil
				_ = k8sClient.Update(ctx, db)
				_ = k8sClient.Delete(ctx, db)
			}
		})

        It("should create, update, and delete a Database", func() {
			By("Creating a Database resource")
            db := &databasev1.Database{
                ObjectMeta: metav1.ObjectMeta{
					Name:      dbName,
                    Namespace: "default",
                },
                Spec: databasev1.DatabaseSpec{
					Image:        "postgres:14",
					Replicas:     ptr.To(int32(1)),
                    DatabaseName: "mydb",
					Username:     "admin",
                    Storage: databasev1.StorageSpec{
						Size: "1Gi",
                    },
                },
            }
            Expect(k8sClient.Create(ctx, db)).To(Succeed())
            
			By("Waiting for StatefulSet to be created")
            Eventually(func() error {
                ss := &appsv1.StatefulSet{}
                return k8sClient.Get(ctx, key, ss)
            }, timeout, interval).Should(Succeed())
            
			By("Verifying StatefulSet has correct initial spec")
                ss := &appsv1.StatefulSet{}
			Expect(k8sClient.Get(ctx, key, ss)).To(Succeed())
			Expect(*ss.Spec.Replicas).To(Equal(int32(1)))
            
			By("Updating Database replicas to 3")
            Expect(k8sClient.Get(ctx, key, db)).To(Succeed())
			db.Spec.Replicas = ptr.To(int32(3))
            Expect(k8sClient.Update(ctx, db)).To(Succeed())
            
			By("Waiting for StatefulSet replicas to be updated to 3")
			Eventually(func() int32 {
                ss := &appsv1.StatefulSet{}
				if err := k8sClient.Get(ctx, key, ss); err != nil {
					return 0
				}
				return *ss.Spec.Replicas
			}, timeout, interval).Should(Equal(int32(3)))
            
			By("Deleting the Database")
            Expect(k8sClient.Delete(ctx, db)).To(Succeed())
            
			By("Verifying the Database is deleted")
            Eventually(func() bool {
                err := k8sClient.Get(ctx, key, db)
                return client.IgnoreNotFound(err) == nil
            }, timeout, interval).Should(BeTrue())
        })

		It("should create all child resources", func() {
			By("Creating a Database resource")
			db := &databasev1.Database{
				ObjectMeta: metav1.ObjectMeta{
					Name:      dbName,
					Namespace: "default",
				},
				Spec: databasev1.DatabaseSpec{
					Image:        "postgres:14",
					Replicas:     ptr.To(int32(1)),
					DatabaseName: "mydb",
					Username:     "admin",
					Storage: databasev1.StorageSpec{
						Size: "1Gi",
					},
				},
			}
			Expect(k8sClient.Create(ctx, db)).To(Succeed())

			By("Verifying the StatefulSet was created")
			Eventually(func() error {
				ss := &appsv1.StatefulSet{}
				return k8sClient.Get(ctx, key, ss)
			}, timeout, interval).Should(Succeed())

			By("Verifying the Service was created")
			Eventually(func() error {
				svc := &corev1.Service{}
				return k8sClient.Get(ctx, key, svc)
			}, timeout, interval).Should(Succeed())

			By("Verifying the Secret was created")
			secretKey := types.NamespacedName{
				Name:      fmt.Sprintf("%s-credentials", dbName),
				Namespace: "default",
			}
			Eventually(func() error {
				secret := &corev1.Secret{}
				return k8sClient.Get(ctx, secretKey, secret)
			}, timeout, interval).Should(Succeed())
		})
    })
})
```

**Ключевые особенности:**
- Использует уникальные имена ресурсов для избежания конфликтов тестов
- Правильная очистка в `AfterEach` (удаляет финализаторы перед удалением)
- Тестирует полный жизненный цикл: создание → обновление → удаление
- Тестирует масштабирование (реплики 1 → 3)
- Тестирует создание дочерних ресурсов (StatefulSet, Service, Secret)

## Упражнение 3: тестирование вебхуков (опционально)

**Примечание**: тесты вебхуков требуют, чтобы вебхуки были развёрнуты и настроены с cert-manager. Если вы не настроили вебхуки, пропустите это упражнение.

### Задача 3.1: протестируйте валидирующий вебхук

Добавьте в `test/integration/database_test.go` (внутри основного блока Describe):

```go
	// Only run if webhooks are deployed
Context("Validating webhook", func() {
    It("should reject invalid Database", func() {
        db := &databasev1.Database{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "invalid-db",
                Namespace: "default",
            },
            Spec: databasev1.DatabaseSpec{
					Image:        "nginx:latest", // Invalid: not PostgreSQL
                DatabaseName: "mydb",
					Username:     "admin",
                Storage: databasev1.StorageSpec{
                    Size: "10Gi",
                },
            },
        }
        
        err := k8sClient.Create(ctx, db)
        Expect(err).To(HaveOccurred())
        Expect(err.Error()).To(ContainSubstring("must be a PostgreSQL image"))
    })
    
    It("should accept valid Database", func() {
        db := &databasev1.Database{
            ObjectMeta: metav1.ObjectMeta{
                Name:      "valid-db",
                Namespace: "default",
            },
            Spec: databasev1.DatabaseSpec{
					Image:        "postgres:14",
                DatabaseName: "mydb",
					Username:     "admin",
                Storage: databasev1.StorageSpec{
                    Size: "10Gi",
                },
            },
        }
        
        Expect(k8sClient.Create(ctx, db)).To(Succeed())

			// Cleanup
			Expect(k8sClient.Delete(ctx, db)).To(Succeed())
    })
})
```

**Примечание**: если вебхуки не развёрнуты, тест «отклонить некорректный» не пройдёт, потому что валидация происходит только в вебхуке. Вы можете пропустить тесты вебхуков командой:

```bash
ginkgo -v -skip="webhook" ./test/integration
```

## Упражнение 4: запуск интеграционных тестов

### Задача 4.1: запустите тесты локально

```bash
# Ensure kind cluster is running, if not use ./scripts/setup-kind-cluster.sh
kind get clusters

# Deploy
# For Docker:
make deploy IMG=postgres-operator:latest

# For Podman:
make deploy IMG=localhost/postgres-operator:latest

# Run integration tests
ginkgo -v ./test/integration
```

### Задача 4.2: запуск с фокусом

```bash
# Run specific test
ginkgo -v -focus="Database lifecycle" ./test/integration
```

## Упражнение 5: интеграция с CI/CD

### Задача 5.1: создайте workflow GitHub Actions

Создайте `.github/workflows/integration-tests.yml`:

```yaml
name: Integration Tests

on: [push, pull_request]

env:
  IMG: postgres-operator:ci

jobs:
  integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.24'
      
      - name: Install kind
        run: |
          go install sigs.k8s.io/kind@latest
      
      - name: Install ginkgo
        run: |
          go install github.com/onsi/ginkgo/v2/ginkgo@latest
      
      - name: Create cluster
        run: kind create cluster --image kindest/node:v1.32.0 --wait 60s
      
      - name: Build Docker image
        run: make docker-build IMG=${{ env.IMG }}
      
      - name: Load image into kind
        run: kind load docker-image ${{ env.IMG }}
      
      - name: Deploy operator (includes cert-manager)
        run: make deploy IMG=${{ env.IMG }}
      
      - name: Wait for cert-manager
        run: |
          kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=120s
          kubectl wait --for=condition=Available deployment/cert-manager-webhook -n cert-manager --timeout=120s
      
      - name: Wait for operator
        run: |
          kubectl wait --for=condition=ready pod -l control-plane=controller-manager -n postgres-operator-system --timeout=120s
      
      - name: Run integration tests
        run: |
          ginkgo -v ./test/integration
      
      - name: Debug on failure
        if: failure()
        run: |
          echo "=== Pods in all namespaces ==="
          kubectl get pods -A
          echo "=== Operator logs ==="
          kubectl logs -n postgres-operator-system -l control-plane=controller-manager --tail=100 || true
          echo "=== Events ==="
          kubectl get events -n postgres-operator-system --sort-by='.lastTimestamp' || true
      
      - name: Cleanup
        if: always()
        run: kind delete cluster
```

**Ключевые моменты:**
- **Сначала соберите образ** — `make docker-build` создаёт образ контейнера
- **Загрузите в kind** — `kind load docker-image` делает образ доступным для кластера
- **Указывайте пространство имён** — `-n postgres-operator-system` в kubectl wait
- **Дождитесь cert-manager** — cert-manager должен быть готов до запуска оператора (вебхукам нужны TLS-сертификаты)
- **Отладка при сбое** — логи помогают диагностировать проблемы

## Очистка

```bash
# Clean up test resources
kubectl delete databases --all

# Clean up cluster (if needed)
kind delete cluster
```

## Итоги лабораторной

В этой лабораторной вы:
- Настроили среду интеграционного тестирования
- Написали сквозные тесты
- Протестировали полные рабочие процессы
- Протестировали вебхуки
- Интегрировали с CI/CD

## Ключевые уроки

1. **Регистрируйте пользовательские типы в схеме** — клиент k8s должен знать о ваших типах CRD через `databasev1.AddToScheme(scheme.Scheme)`
2. **Передавайте схему клиенту** — используйте `client.Options{Scheme: scheme.Scheme}` при создании клиента
3. **Интеграционные тесты используют реальные кластеры** — тесты выполняются на настоящем API Kubernetes (kind, minikube и т. д.)
4. **Eventually ожидает асинхронные операции** — контроллеры асинхронны; используйте `Eventually` для утверждений
5. **Тестируйте полные рабочие процессы** — жизненный цикл создание → обновление → удаление
6. **Вебхуки требуют развёртывания** — тесты вебхуков работают только когда вебхуки развёрнуты с cert-manager
7. **CI/CD автоматизирует тестирование** — используйте GitHub Actions или аналог для автоматизированного тестирования
8. **Очищайте ресурсы после тестов** — удаляйте созданные ресурсы, чтобы избежать «загрязнения» тестов

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Integration Test Examples](../solutions/integration_test.go) — полные примеры интеграционных тестов

## Дальнейшие шаги

Теперь давайте добавим наблюдаемость и изучим приёмы отладки!

**Навигация:** [← Предыдущая лабораторная: Модульное тестирование](lab-02-unit-testing-envtest.md) | [Связанный урок](../lessons/03-integration-testing.md) | [Следующая лабораторная: Наблюдаемость →](lab-04-debugging-observability.md)
