---
layout: default
title: "06.2 Unit Testing Envtest"
nav_order: 2
parent: "Модуль 6: Тестирование и отладка"
grand_parent: Модули
mermaid: true
---

# Урок 6.2: Модульное тестирование с envtest

**Навигация:** [← Предыдущий: Основы тестирования](01-testing-fundamentals.md) | [Обзор модуля](../README.md) | [Далее: Интеграционное тестирование →](03-integration-testing.md)

## Введение

Для модульного тестирования операторов нужен API-сервер Kubernetes, но полноценный кластер не требуется. **envtest** предоставляет легковесный API-сервер Kubernetes, специально предназначенный для тестирования. Этот урок научит вас использовать envtest для написания исчерпывающих модульных тестов ваших операторов.

## Что такое envtest?

envtest предоставляет минимальный API-сервер Kubernetes:

```mermaid
graph TB
    ENVTEST[envtest]
    
    ENVTEST --> API[Kubernetes API Server]
    ENVTEST --> ETCD[etcd]
    
    API --> CRUD[CRUD Operations]
    API --> WATCH[Watch Operations]
    API --> VALIDATION[Validation]
    
    ETCD --> STORAGE[Storage]
    
    style ENVTEST fill:#90EE90
```

**Возможности:**
- Без kubelet, без scheduler
- Реальный API Kubernetes
- Быстрый запуск
- Изолированная среда

## Процесс настройки envtest

Вот как работает envtest:

```mermaid
sequenceDiagram
    participant Test
    participant Envtest
    participant API as API Server
    participant etcd as etcd
    
    Test->>Envtest: StartEnvironment
    Envtest->>API: Start API Server
    Envtest->>etcd: Start etcd
    API->>etcd: Connect
    API-->>Test: Ready
    Test->>API: Create Resources
    API->>etcd: Store
    Test->>API: Get Resources
    API-->>Test: Resources
    Test->>Envtest: StopEnvironment
```

## Настройка envtest

### Шаг 1: установите зависимости

```bash
# Install envtest binaries
go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest

# Download envtest binaries
setup-envtest use
```

### Шаг 2: создайте настройку теста

```go
package controller

import (
    "path/filepath"
    "testing"
    
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    
    "k8s.io/client-go/kubernetes/scheme"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
)

var (
    k8sClient client.Client
    testEnv   *envtest.Environment
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    By("bootstrapping test environment")
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }
    
    cfg, err := testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())
    
    err = databasev1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())
    
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())
    Expect(k8sClient).NotTo(BeNil())
})

var _ = AfterSuite(func() {
    By("tearing down the test environment")
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

## Написание модульных тестов

### Пример: тестирование согласования

```go
var _ = Describe("DatabaseReconciler", func() {
    var (
        ctx    context.Context
        cancel context.CancelFunc
    )
    
    BeforeEach(func() {
        ctx, cancel = context.WithCancel(context.Background())
    })
    
    AfterEach(func() {
        cancel()
    })
    
    Context("When reconciling a Database", func() {
        It("should create a StatefulSet", func() {
            // Arrange
            db := &databasev1.Database{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      "test-db",
                    Namespace: "default",
                },
                Spec: databasev1.DatabaseSpec{
                    Image:       "postgres:14",
                    Replicas:    pointer.Int32(1),
                    DatabaseName: "mydb",
                    Username:    "admin",
                    Storage: databasev1.StorageSpec{
                        Size: "10Gi",
                    },
                },
            }
            Expect(k8sClient.Create(ctx, db)).To(Succeed())
            
            // Act
            reconciler := &DatabaseReconciler{
                Client: k8sClient,
                Scheme: scheme.Scheme,
            }
            _, err := reconciler.Reconcile(ctx, ctrl.Request{
                NamespacedName: types.NamespacedName{
                    Name:      "test-db",
                    Namespace: "default",
                },
            })
            
            // Assert
            Expect(err).NotTo(HaveOccurred())
            
            statefulSet := &appsv1.StatefulSet{}
            Expect(k8sClient.Get(ctx, types.NamespacedName{
                Name:      "test-db",
                Namespace: "default",
            }, statefulSet)).To(Succeed())
            
            Expect(statefulSet.Spec.Replicas).To(Equal(pointer.Int32(1)))
            Expect(statefulSet.Spec.Template.Spec.Containers[0].Image).To(Equal("postgres:14"))
        })
    })
})
```

## Тесты, управляемые таблицей, с envtest

```go
Describe("Database validation", func() {
    tests := []struct {
        name    string
        db      *databasev1.Database
        wantErr bool
    }{
        {
            name: "valid database",
            db: &databasev1.Database{
                Spec: databasev1.DatabaseSpec{
                    Image:       "postgres:14",
                    DatabaseName: "mydb",
                    Username:    "admin",
                    Storage: databasev1.StorageSpec{
                        Size: "10Gi",
                    },
                },
            },
            wantErr: false,
        },
        {
            name: "missing image",
            db: &databasev1.Database{
                Spec: databasev1.DatabaseSpec{
                    DatabaseName: "mydb",
                    Username:    "admin",
                    Storage: databasev1.StorageSpec{
                        Size: "10Gi",
                    },
                },
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        It(tt.name, func() {
            err := k8sClient.Create(ctx, tt.db)
            if tt.wantErr {
                Expect(err).To(HaveOccurred())
            } else {
                Expect(err).NotTo(HaveOccurred())
            }
        })
    }
})
```

## Тестирование случаев ошибок

```go
Context("When StatefulSet creation fails", func() {
    It("should return an error", func() {
        // Create Database with invalid spec
        db := &databasev1.Database{
            Spec: databasev1.DatabaseSpec{
                // Invalid: missing required fields
            },
        }
        
        reconciler := &DatabaseReconciler{
            Client: k8sClient,
            Scheme: scheme.Scheme,
        }
        
        _, err := reconciler.Reconcile(ctx, ctrl.Request{
            NamespacedName: types.NamespacedName{
                Name:      "test-db",
                Namespace: "default",
            },
        })
        
        Expect(err).To(HaveOccurred())
    })
})
```

## Тестирование обновлений

```go
Context("When updating a Database", func() {
    It("should update the StatefulSet", func() {
        // Create initial Database
        db := &databasev1.Database{...}
        Expect(k8sClient.Create(ctx, db)).To(Succeed())
        
        // Reconcile to create StatefulSet
        reconciler.Reconcile(ctx, req)
        
        // Update Database
        Expect(k8sClient.Get(ctx, key, db)).To(Succeed())
        db.Spec.Replicas = pointer.Int32(3)
        Expect(k8sClient.Update(ctx, db)).To(Succeed())
        
        // Reconcile again
        reconciler.Reconcile(ctx, req)
        
        // Verify StatefulSet updated
        statefulSet := &appsv1.StatefulSet{}
        Expect(k8sClient.Get(ctx, key, statefulSet)).To(Succeed())
        Expect(statefulSet.Spec.Replicas).To(Equal(pointer.Int32(3)))
    })
})
```

## Ключевые выводы

- **envtest** предоставляет легковесный API Kubernetes для тестирования
- **Настройка** включает запуск тестовой среды и создание клиента
- **Пишите тесты** с использованием Ginkgo/Gomega для структуры
- **Тестируйте логику** согласования с реальным API
- **Используйте тесты, управляемые таблицей**, для множества сценариев
- **Тестируйте случаи ошибок** и граничные случаи
- **Тестируйте обновления** и изменения состояния
- **Очищайте** ресурсы в AfterEach

## Что нужно понимать для создания операторов

При написании модульных тестов:
- Используйте envtest для API Kubernetes
- Настраивайте тестовую среду в BeforeSuite
- Очищайте в AfterSuite
- Тестируйте все пути согласования
- Тестируйте случаи ошибок
- Используйте тесты, управляемые таблицей, для множества сценариев
- Проверяйте создание/обновление ресурсов
- Тестируйте переходы состояний

## Связанная лабораторная работа

- [Лабораторная 6.2: Написание модульных тестов](../labs/lab-02-unit-testing-envtest.md) — практические упражнения для этого урока

## Источники

### Официальная документация
- [Документация envtest](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest)
- [Руководство по тестированию Kubebuilder](https://book.kubebuilder.io/cronjob-tutorial/writing-tests.html)
- [Пакет testing в Go](https://pkg.go.dev/testing)

### Дополнительное чтение
- **Kubernetes Operators**, Jason Dobies и Joshua Wood — глава 10: Testing
- **Programming Kubernetes**, Michael Hausenblas и Stefan Schimanski — глава 10: Testing
- [Лучшие практики тестирования на Go](https://golang.org/doc/effective_go#testing)

### Смежные темы
- [Тесты, управляемые таблицей](https://go.dev/wiki/TableDrivenTests)
- [Покрытие тестами](https://go.dev/blog/cover)
- [Бенчмаркинг в Go](https://golang.org/pkg/testing/#hdr-Benchmarks)

## Дальнейшие шаги

Теперь, когда вы понимаете модульное тестирование, давайте изучим интеграционное тестирование с реальными кластерами.

**Навигация:** [← Предыдущий: Основы тестирования](01-testing-fundamentals.md) | [Обзор модуля](../README.md) | [Далее: Интеграционное тестирование →](03-integration-testing.md)
