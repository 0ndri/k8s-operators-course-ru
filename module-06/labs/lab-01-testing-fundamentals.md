---
layout: default
title: "Lab 06.1: Testing Fundamentals"
nav_order: 11
parent: "Модуль 6: Тестирование и отладка"
grand_parent: Модули
mermaid: true
---

# Лабораторная 6.1: Настройка среды тестирования

**Связанный урок:** [Урок 6.1: Основы тестирования](../lessons/01-testing-fundamentals.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: Модульное тестирование →](lab-02-unit-testing-envtest.md)

## Цели

- Настроить инструменты и зависимости для тестирования
- Понять структуру тестов
- Создать каркас тестов
- Подготовиться к написанию тестов

## Предварительные требования

- Завершение [Модуля 5](../../module-05/README.md)
- Оператор Database из Модулей 3/4/5
- Установленный Go 1.24+
- Понимание тестирования на Go

## Упражнение 1: установка инструментов тестирования

### Задача 1.1: установите Ginkgo и Gomega

```bash
# Install Ginkgo
go install github.com/onsi/ginkgo/v2/ginkgo@latest

# Install Gomega
go get github.com/onsi/gomega/...

# Verify installation
ginkgo version
```

### Задача 1.2: установите инструменты envtest

```bash
# Install setup-envtest
go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest

# Download envtest binaries
setup-envtest use

# Verify
setup-envtest list
```

### Задача 1.3: установите отладчик Delve

```bash
# Install Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Verify installation
dlv version
```

## Упражнение 2: настройка структуры тестов

### Задача 2.1: перейдите к вашему оператору

```bash
# Navigate to your operator
cd ~/postgres-operator
```

Когда вы запускаете `kubebuilder create api` с `--resource --controller`, Kubebuilder автоматически генерирует файлы каркаса тестов в `internal/controller/`:
- `suite_test.go` — настройка набора тестов с envtest
- `<resource>_controller_test.go` — базовый тест контроллера

### Задача 2.2: изучите сгенерированный файл набора тестов

Сгенерированный `internal/controller/suite_test.go` имеет такую структуру:

```go
package controller

import (
    "context"
    "os"
    "path/filepath"
    "testing"
    
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
    // +kubebuilder:scaffold:imports
)

// These tests use Ginkgo (BDD-style Go testing framework). Refer to
// http://onsi.github.io/ginkgo/ to learn more about Ginkgo.

var (
    ctx       context.Context
    cancel    context.CancelFunc
    testEnv   *envtest.Environment
    cfg       *rest.Config
    k8sClient client.Client
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)

    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))

    ctx, cancel = context.WithCancel(context.TODO())

    var err error
    err = databasev1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())

    // +kubebuilder:scaffold:scheme

    By("bootstrapping test environment")
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }
    
    // Retrieve the first found binary directory to allow running tests from IDEs
    if getFirstFoundEnvTestBinaryDir() != "" {
        testEnv.BinaryAssetsDirectory = getFirstFoundEnvTestBinaryDir()
    }

    // cfg is defined in this file globally.
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())
    
    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())
    Expect(k8sClient).NotTo(BeNil())
})

var _ = AfterSuite(func() {
    By("tearing down the test environment")
    cancel()
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})

// getFirstFoundEnvTestBinaryDir locates the first binary in the specified path.
// ENVTEST-based tests depend on specific binaries, usually located in paths set by
// controller-runtime. When running tests directly (e.g., via an IDE) without using
// Makefile targets, the 'BinaryAssetsDirectory' must be explicitly configured.
//
// This function streamlines the process by finding the required binaries, similar to
// setting the 'KUBEBUILDER_ASSETS' environment variable. To ensure the binaries are
// properly set up, run 'make setup-envtest' beforehand.
func getFirstFoundEnvTestBinaryDir() string {
    basePath := filepath.Join("..", "..", "bin", "k8s")
    entries, err := os.ReadDir(basePath)
    if err != nil {
        logf.Log.Error(err, "Failed to read directory", "path", basePath)
        return ""
    }
    for _, entry := range entries {
        if entry.IsDir() {
            return filepath.Join(basePath, entry.Name())
        }
    }
    return ""
}
```

**Ключевые особенности сгенерированного набора:**
- **Контекст на уровне пакета**: `ctx` и `cancel` доступны всем тестам
- **Поддержка IDE**: `getFirstFoundEnvTestBinaryDir()` находит бинарники envtest для запуска из IDE
- **Логирование**: настроено с логгером zap, пишущим в GinkgoWriter
- **Маркеры каркаса**: `// +kubebuilder:scaffold:imports` и `// +kubebuilder:scaffold:scheme` для будущих добавлений API

## Упражнение 3: изучение сгенерированного теста контроллера

### Задача 3.1: разберитесь в структуре сгенерированного теста

Сгенерированный `internal/controller/database_controller_test.go` имеет такую структуру:

```go
package controller

import (
    "context"
    
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    
    databasev1 "github.com/example/postgres-operator/api/v1"
)

var _ = Describe("Database Controller", func() {
    Context("When reconciling a resource", func() {
        const resourceName = "test-resource"

        ctx := context.Background()

        typeNamespacedName := types.NamespacedName{
            Name:      resourceName,
            Namespace: "default", // TODO(user):Modify as needed
        }
        database := &databasev1.Database{}
    
    BeforeEach(func() {
            By("creating the custom resource for the Kind Database")
            err := k8sClient.Get(ctx, typeNamespacedName, database)
            if err != nil && errors.IsNotFound(err) {
                resource := &databasev1.Database{
                    ObjectMeta: metav1.ObjectMeta{
                        Name:      resourceName,
                        Namespace: "default",
                    },
                    // TODO(user): Specify other spec details if needed.
                }
                Expect(k8sClient.Create(ctx, resource)).To(Succeed())
            }
    })
    
    AfterEach(func() {
            // TODO(user): Cleanup logic after each test, like removing the resource instance.
            resource := &databasev1.Database{}
            err := k8sClient.Get(ctx, typeNamespacedName, resource)
            Expect(err).NotTo(HaveOccurred())

            By("Cleanup the specific resource instance Database")
            Expect(k8sClient.Delete(ctx, resource)).To(Succeed())
        })
        
        It("should successfully reconcile the resource", func() {
            By("Reconciling the created resource")
            controllerReconciler := &DatabaseReconciler{
                Client: k8sClient,
                Scheme: k8sClient.Scheme(),
            }

            _, err := controllerReconciler.Reconcile(ctx, reconcile.Request{
                NamespacedName: typeNamespacedName,
            })
            Expect(err).NotTo(HaveOccurred())
            // TODO(user): Add more specific assertions depending on your controller's reconciliation logic.
            // Example: If you expect a certain status condition after reconciliation, verify it here.
        })
    })
})
```

**Ключевые особенности сгенерированного теста:**
- **Настройка/очистка ресурса**: `BeforeEach` создаёт ресурс, `AfterEach` удаляет его
- **Прямой вызов реконсайлера**: создаёт `DatabaseReconciler` и вызывает `Reconcile()` напрямую
- **Паттерн NamespacedName**: использует `types.NamespacedName` для идентификации ресурса
- **Маркеры TODO**: указывают, где нужно доработать под ваш конкретный контроллер
- **Использует переменные уровня пакета**: обращается к `k8sClient` из `suite_test.go`

## Упражнение 4: запуск тестов

### Задача 4.1: запустите тесты

```bash
# Setup envtest binaries first
make setup-envtest

# Run all tests using make (recommended)
make test

# Or run tests directly with go test
go test ./internal/controller/...

# Run with Ginkgo (verbose)
ginkgo -v ./internal/controller/...

# Run specific test
ginkgo -v -focus="Database Controller" ./internal/controller/...
```

### Задача 4.2: проверьте покрытие тестами

```bash
# Run with coverage
go test -cover ./internal/controller/...

# Generate coverage report
go test -coverprofile=coverage.out ./internal/controller/...
go tool cover -html=coverage.out
```

## Упражнение 5: проверка настройки

### Задача 5.1: проверьте все инструменты

```bash
# Check Ginkgo
ginkgo version

# Check envtest
setup-envtest list

# Check Delve
dlv version

# Check Go
go version
```

## Очистка

```bash
# Clean up test resources (if any)
# Tests should clean up automatically
```

## Итоги лабораторной

В этой лабораторной вы:
- Установили инструменты тестирования (Ginkgo, Gomega, envtest, Delve)
- Изучили структуру каркаса тестов, сгенерированного Kubebuilder
- Разобрались в настройке набора тестов с envtest
- Изучили паттерн теста контроллера
- Запустили тесты и проверили покрытие

## Ключевые уроки

1. **Kubebuilder генерирует каркас тестов** — когда вы создаёте API с `--controller`, тестовые файлы генерируются автоматически
2. **Ginkgo обеспечивает структуру тестов в стиле BDD** — блоки Describe/Context/It организуют тесты
3. **envtest предоставляет легковесный API Kubernetes** — для тестов контроллера не нужен полноценный кластер
4. **Настройка набора в BeforeSuite/AfterSuite** — среда инициализируется один раз на набор тестов
5. **Переменные уровня пакета** — `ctx`, `k8sClient`, `cfg` разделяются между тестами
6. **Встроенная поддержка IDE** — `getFirstFoundEnvTestBinaryDir()` позволяет запускать тесты из IDE
7. **Прямой вызов реконсайлера** — тесты вызывают `Reconcile()` напрямую для детерминированных результатов
8. **Маркеры каркаса** — комментарии `// +kubebuilder:scaffold:*` позволяют добавлять API в будущем

## Решения

Настройка набора тестов из этой лабораторной соответствует каркасу, сгенерированному Kubebuilder:
- [Test Suite Setup](../solutions/suite_test.go) — полный набор тестов с конфигурацией envtest
- [Controller Test](../solutions/database_controller_test.go) — базовая структура теста контроллера

## Дальнейшие шаги

Теперь давайте напишем исчерпывающие модульные тесты для вашего оператора!

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-testing-fundamentals.md) | [Следующая лабораторная: Модульное тестирование →](lab-02-unit-testing-envtest.md)
