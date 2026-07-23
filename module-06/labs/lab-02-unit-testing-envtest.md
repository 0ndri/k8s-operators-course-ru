---
layout: default
title: "Lab 06.2: Unit Testing Envtest"
nav_order: 12
parent: "Модуль 6: Тестирование и отладка"
grand_parent: Модули
mermaid: true
---

# Лабораторная 6.2: Написание модульных тестов

**Связанный урок:** [Урок 6.2: Модульное тестирование с envtest](../lessons/02-unit-testing-envtest.md)  
**Навигация:** [← Предыдущая лабораторная: Основы тестирования](lab-01-testing-fundamentals.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Интеграционное тестирование →](lab-03-integration-testing.md)

## Цели

- Написать модульные тесты для логики согласования
- Протестировать создание и обновление ресурсов
- Протестировать случаи ошибок
- Понять паттерны тестирования конечного автомата
- Достичь хорошего покрытия тестами

## Предварительные требования

- Завершение [Лабораторной 6.1](lab-01-testing-fundamentals.md)
- Настроенная среда тестирования
- Готовый оператор Database

## Понимание контроллера

Перед написанием тестов учтите, что `DatabaseReconciler` использует **паттерн конечного автомата** с фазами:
- `Pending` → `Provisioning` → `Configuring` → `Deploying` → `Verifying` → `Ready`

Каждый вызов `Reconcile()` продвигает состояние на одну фазу. Это означает, что для полного развёртывания базы данных нужно несколько вызовов reconcile.

## Упражнение 1: тестирование базового согласования

### Задача 1.1: протестируйте начальный переход состояния

Обновите `internal/controller/database_controller_test.go`, добавив новый Context теста. Обратите внимание, как мы используем уникальные имена ресурсов с `GenerateName`, чтобы избежать конфликтов между тестами:

```go
Context("When reconciling a new Database", func() {
    var (
        resourceName      string
        typeNamespacedName types.NamespacedName
    )

    BeforeEach(func() {
        // Generate unique name for each test
        resourceName = fmt.Sprintf("test-db-%d", time.Now().UnixNano())
        typeNamespacedName = types.NamespacedName{
            Name:      resourceName,
            Namespace: "default",
        }

        // Create the Database resource
        resource := &databasev1.Database{
            ObjectMeta: metav1.ObjectMeta{
                Name:      resourceName,
                Namespace: "default",
            },
            Spec: databasev1.DatabaseSpec{
                Image:        "postgres:14",
                Replicas:     ptr.To(int32(1)),
                DatabaseName: "testdb",
                Username:     "testuser",
                Storage: databasev1.StorageSpec{
                    Size: "1Gi",
                },
            },
        }
        Expect(k8sClient.Create(ctx, resource)).To(Succeed())
    })

    AfterEach(func() {
        // Cleanup
        resource := &databasev1.Database{}
        err := k8sClient.Get(ctx, typeNamespacedName, resource)
        if err == nil {
            // Remove finalizer to allow deletion
            resource.Finalizers = nil
            _ = k8sClient.Update(ctx, resource)
            _ = k8sClient.Delete(ctx, resource)
        }
    })

    It("should transition from Pending to Provisioning", func() {
        By("Reconciling the created resource")
        controllerReconciler := &DatabaseReconciler{
            Client: k8sClient,
            Scheme: k8sClient.Scheme(),
        }

        // First reconcile: Pending -> Provisioning
        _, err := controllerReconciler.Reconcile(ctx, reconcile.Request{
            NamespacedName: typeNamespacedName,
        })
        Expect(err).NotTo(HaveOccurred())

        // Verify status was updated
        db := &databasev1.Database{}
        Expect(k8sClient.Get(ctx, typeNamespacedName, db)).To(Succeed())
        Expect(db.Status.Phase).To(Equal("Provisioning"))
        Expect(db.Status.Ready).To(BeFalse())
    })
})
```

**Необходимые импорты** (добавьте в блок import):

```go
import (
    "context"
    "fmt"
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/types"
    "k8s.io/utils/ptr"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"

    databasev1 "github.com/example/postgres-operator/api/v1"
)
```

## Упражнение 2: тестирование создания ресурсов через конечный автомат

### Задача 2.1: протестируйте создание StatefulSet

StatefulSet создаётся во время фазы `Provisioning`. Протестируйте это, выполнив несколько вызовов reconcile:

```go
Context("When progressing through provisioning", func() {
    var (
        resourceName       string
        typeNamespacedName types.NamespacedName
    )

    BeforeEach(func() {
        resourceName = fmt.Sprintf("test-provision-%d", time.Now().UnixNano())
        typeNamespacedName = types.NamespacedName{
            Name:      resourceName,
            Namespace: "default",
        }

        resource := &databasev1.Database{
            ObjectMeta: metav1.ObjectMeta{
                Name:      resourceName,
                Namespace: "default",
            },
            Spec: databasev1.DatabaseSpec{
                Image:        "postgres:14",
                Replicas:     ptr.To(int32(1)),
                DatabaseName: "testdb",
                Username:     "testuser",
                Storage: databasev1.StorageSpec{
                    Size: "1Gi",
                },
            },
        }
        Expect(k8sClient.Create(ctx, resource)).To(Succeed())
    })

    AfterEach(func() {
        resource := &databasev1.Database{}
        err := k8sClient.Get(ctx, typeNamespacedName, resource)
        if err == nil {
            resource.Finalizers = nil
            _ = k8sClient.Update(ctx, resource)
            _ = k8sClient.Delete(ctx, resource)
        }
    })

    It("should create Secret and StatefulSet", func() {
        reconciler := &DatabaseReconciler{
            Client: k8sClient,
            Scheme: k8sClient.Scheme(),
        }
        req := reconcile.Request{NamespacedName: typeNamespacedName}

        By("First reconcile: Pending -> Provisioning")
        _, err := reconciler.Reconcile(ctx, req)
        Expect(err).NotTo(HaveOccurred())

        By("Second reconcile: Creates Secret and StatefulSet")
        _, err = reconciler.Reconcile(ctx, req)
        Expect(err).NotTo(HaveOccurred())

        By("Verifying Secret was created")
        secret := &corev1.Secret{}
        secretName := fmt.Sprintf("%s-credentials", resourceName)
        Expect(k8sClient.Get(ctx, types.NamespacedName{
            Name:      secretName,
            Namespace: "default",
        }, secret)).To(Succeed())
        Expect(secret.Data).To(HaveKey("username"))
        Expect(secret.Data).To(HaveKey("password"))

        By("Verifying StatefulSet was created")
        statefulSet := &appsv1.StatefulSet{}
        Expect(k8sClient.Get(ctx, typeNamespacedName, statefulSet)).To(Succeed())
        Expect(*statefulSet.Spec.Replicas).To(Equal(int32(1)))
        Expect(statefulSet.Spec.Template.Spec.Containers[0].Image).To(Equal("postgres:14"))
    })
})
```

## Упражнение 3: тестирование случаев ошибок

### Задача 3.1: протестируйте отсутствующий ресурс

```go
Context("When Database is not found", func() {
    It("should not return an error", func() {
        reconciler := &DatabaseReconciler{
            Client: k8sClient,
            Scheme: k8sClient.Scheme(),
        }

        req := reconcile.Request{
            NamespacedName: types.NamespacedName{
                Name:      "non-existent-database",
                Namespace: "default",
            },
        }

        result, err := reconciler.Reconcile(ctx, req)
        Expect(err).NotTo(HaveOccurred())
        Expect(result.Requeue).To(BeFalse())
        Expect(result.RequeueAfter).To(Equal(time.Duration(0)))
    })
})
```

### Задача 3.2: протестируйте добавление финализатора

```go
var _ = Describe("Database validation", func() {
    var (
        ctx               context.Context
        typeNamespacedName types.NamespacedName
    )

    BeforeEach(func() {
        ctx = context.Background()
        typeNamespacedName = types.NamespacedName{
            Name:      "test-database",
            Namespace: "default",
        }
        
        // Create the database resource
        resource := &databasev1.Database{
            ObjectMeta: metav1.ObjectMeta{
                Name:      typeNamespacedName.Name,
                Namespace: typeNamespacedName.Namespace,
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
        Expect(k8sClient.Create(ctx, resource)).To(Succeed())
    })

    AfterEach(func() {
        resource := &databasev1.Database{}
        err := k8sClient.Get(ctx, typeNamespacedName, resource)
        if err == nil {
            resource.Finalizers = nil
            _ = k8sClient.Update(ctx, resource)
            _ = k8sClient.Delete(ctx, resource)
        }
    })

    It("should add finalizer on first reconcile", func() {
        reconciler := &DatabaseReconciler{
            Client: k8sClient,
            Scheme: k8sClient.Scheme(),
        }

        _, err := reconciler.Reconcile(ctx, reconcile.Request{
            NamespacedName: typeNamespacedName,
        })
        Expect(err).NotTo(HaveOccurred())

        db := &databasev1.Database{}
        Expect(k8sClient.Get(ctx, typeNamespacedName, db)).To(Succeed())
        Expect(db.Finalizers).To(ContainElement("database.example.com/finalizer"))
    })
})
```

## Упражнение 4: тестирование создания Service

### Задача 4.1: протестируйте создание Service в фазе Configuring

```go
Context("When in Configuring phase", func() {
    var (
        resourceName       string
        typeNamespacedName types.NamespacedName
    )

    BeforeEach(func() {
        resourceName = fmt.Sprintf("test-service-%d", time.Now().UnixNano())
        typeNamespacedName = types.NamespacedName{
            Name:      resourceName,
            Namespace: "default",
        }

        resource := &databasev1.Database{
            ObjectMeta: metav1.ObjectMeta{
                Name:      resourceName,
                Namespace: "default",
            },
            Spec: databasev1.DatabaseSpec{
                Image:        "postgres:14",
                DatabaseName: "testdb",
                Username:     "testuser",
                Storage: databasev1.StorageSpec{
                    Size: "1Gi",
                },
            },
        }
        Expect(k8sClient.Create(ctx, resource)).To(Succeed())
    })

    AfterEach(func() {
        resource := &databasev1.Database{}
        err := k8sClient.Get(ctx, typeNamespacedName, resource)
        if err == nil {
            resource.Finalizers = nil
            _ = k8sClient.Update(ctx, resource)
            _ = k8sClient.Delete(ctx, resource)
        }
    })

    It("should create Service", func() {
        reconciler := &DatabaseReconciler{
            Client: k8sClient,
            Scheme: k8sClient.Scheme(),
        }
        req := reconcile.Request{NamespacedName: typeNamespacedName}

        By("Progress through states to Configuring")
        // Pending -> Provisioning
        _, _ = reconciler.Reconcile(ctx, req)
        // Provisioning: creates Secret + StatefulSet, stays in Provisioning
        _, _ = reconciler.Reconcile(ctx, req)
        // Provisioning -> Configuring (StatefulSet exists)
        _, _ = reconciler.Reconcile(ctx, req)
        // Configuring: creates Service
        _, err := reconciler.Reconcile(ctx, req)
        Expect(err).NotTo(HaveOccurred())

        By("Verifying Service was created")
        service := &corev1.Service{}
        Expect(k8sClient.Get(ctx, typeNamespacedName, service)).To(Succeed())
        Expect(service.Spec.Ports[0].Port).To(Equal(int32(5432)))
    })
})
```

## Упражнение 5: покрытие тестами

### Задача 5.1: проверьте покрытие

```bash
# Run tests with coverage
make test

# Or run with coverage profile
go test -coverprofile=coverage.out ./internal/controller/...

# View coverage summary
go tool cover -func=coverage.out

# Generate HTML report
go tool cover -html=coverage.out -o coverage.html
open coverage.html  # macOS
```

### Задача 5.2: улучшите покрытие

Добавьте тесты для:
- Обработки удаления с очисткой финализатора
- Обновлений условий статуса
- Разного количества реплик
- Изменений образа

## Очистка

Блоки `AfterEach` в каждом Context теста обрабатывают очистку автоматически, выполняя:
1. Удаление финализаторов (чтобы разрешить удаление)
2. Удаление тестового ресурса Database

## Итоги лабораторной

В этой лабораторной вы:
- Написали модульные тесты по паттерну каркаса Kubebuilder
- Протестировали переходы конечного автомата
- Протестировали создание ресурсов (Secret, StatefulSet, Service)
- Протестировали случаи ошибок (отсутствующие ресурсы)
- Протестировали добавление финализатора
- Проверили покрытие тестами

## Ключевые уроки

1. **Тестирование конечного автомата** — контроллерам с фазами нужно несколько вызовов reconcile
2. **Используйте уникальные имена ресурсов** — избегайте конфликтов тестов с уникальными именами для каждого теста
3. **Правильная очистка** — удаляйте финализаторы перед удалением в `AfterEach`
4. **Используйте `k8sClient.Scheme()`** — а не `scheme.Scheme` для инициализации реконсайлера
5. **Используйте `reconcile.Request`** — стандартный тип для тестовых запросов
6. **Используйте `k8s.io/utils/ptr`** — для помощников указателей, таких как `ptr.To(int32(1))`
7. **envtest предоставляет реальный API** — тесты выполняются на настоящем API-сервере Kubernetes

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Test Suite Setup](../solutions/suite_test.go) — полный набор тестов с envtest
- [Unit Test Examples](../solutions/database_controller_test.go) — базовая структура теста контроллера

## Дальнейшие шаги

Теперь давайте создадим интеграционные тесты для сквозных сценариев!

**Навигация:** [← Предыдущая лабораторная: Основы тестирования](lab-01-testing-fundamentals.md) | [Связанный урок](../lessons/02-unit-testing-envtest.md) | [Следующая лабораторная: Интеграционное тестирование →](lab-03-integration-testing.md)
