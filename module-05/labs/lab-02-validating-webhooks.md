---
layout: default
title: "Lab 05.2: Validating Webhooks"
nav_order: 12
parent: "Модуль 5: Вебхуки и контроль допуска"
grand_parent: Модули
mermaid: true
---

# Лабораторная 5.2: Создание валидирующего вебхука

**Связанный урок:** [Урок 5.2: Реализация валидирующих вебхуков](../lessons/02-validating-webhooks.md)  
**Навигация:** [← Предыдущая лабораторная: Контроль допуска](lab-01-admission-control.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Мутирующие вебхуки →](lab-03-mutating-webhooks.md)

## Цели

- Сгенерировать каркас валидирующего вебхука с помощью kubebuilder
- Реализовать пользовательскую логику валидации
- Протестировать на корректных и некорректных ресурсах
- Предоставить информативные сообщения об ошибках

## Предварительные требования

- Завершение [Модуля 3](../../module-03/README.md) или [Модуля 4](../../module-04/README.md)
- Проект оператора Database
- Понимание требований к валидации

## Упражнение 1: генерация каркаса валидирующего вебхука

### Задача 1.1: перейдите к вашему оператору

```bash
# Navigate to your Database operator
cd ~/postgres-operator
```

### Задача 1.2: создайте валидирующий вебхук

```bash
# Create validating webhook
kubebuilder create webhook \
  --group database \
  --version v1 \
  --kind Database \
  --programmatic-validation
```

**Обратите внимание:**
- Какие файлы были созданы?
- Что было изменено?

### Задача 1.3: изучите сгенерированный код

```bash
# Check the generated webhook file
cat internal/webhook/v1/database_webhook.go

# Check webhook markers
grep "kubebuilder:webhook" internal/webhook/v1/database_webhook.go
```

**Обратите внимание на структуру:**
- Код вебхука находится в каталоге `internal/webhook/v1/`
- Использует структуру `DatabaseCustomValidator`
- Реализует интерфейс `webhook.CustomValidator`
- Методы принимают `context.Context` первым параметром

## Упражнение 2: реализация логики валидации

### Задача 2.1: добавьте ValidateCreate

Отредактируйте `internal/webhook/v1/database_webhook.go`:

```go
package v1

import (
    "context"
    "fmt"
    "strconv"
    "strings"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"

    databasev1 "github.com/example/postgres-operator/api/v1"
)

var databaselog = logf.Log.WithName("database-resource")

// SetupDatabaseWebhookWithManager registers the webhook for Database in the manager.
func SetupDatabaseWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).For(&databasev1.Database{}).
        WithValidator(&DatabaseCustomValidator{}).
        Complete()
}

// +kubebuilder:webhook:path=/validate-database-example-com-v1-database,mutating=false,failurePolicy=fail,sideEffects=None,groups=database.example.com,resources=databases,verbs=create;update,versions=v1,name=vdatabase-v1.kb.io,admissionReviewVersions=v1

// DatabaseCustomValidator struct is responsible for validating the Database resource
// when it is created, updated, or deleted.
type DatabaseCustomValidator struct {
    // Add more fields as needed for validation
}

var _ webhook.CustomValidator = &DatabaseCustomValidator{}

// ValidateCreate implements webhook.CustomValidator so a webhook will be registered for the type Database.
func (v *DatabaseCustomValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    database, ok := obj.(*databasev1.Database)
    if !ok {
        return nil, fmt.Errorf("expected a Database object but got %T", obj)
    }
    databaselog.Info("Validation for Database upon creation", "name", database.GetName())

    // Validate image is PostgreSQL
    if !strings.Contains(database.Spec.Image, "postgres") {
        return nil, fmt.Errorf("spec.image must be a PostgreSQL image, got %s", database.Spec.Image)
    }

    // Validate replicas and storage relationship
    if database.Spec.Replicas != nil && *database.Spec.Replicas > 5 {
        if database.Spec.Storage.Size == "10Gi" {
            return nil, fmt.Errorf("replicas > 5 requires storage >= 50Gi, got %s", database.Spec.Storage.Size)
        }
    }

    // Validate database name format
    if len(database.Spec.DatabaseName) > 63 {
        return nil, fmt.Errorf("spec.databaseName must be <= 63 characters, got %d", len(database.Spec.DatabaseName))
    }

    return nil, nil
}
```

### Задача 2.2: добавьте ValidateUpdate

```go
// ValidateUpdate implements webhook.CustomValidator so a webhook will be registered for the type Database.
func (v *DatabaseCustomValidator) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
    database, ok := newObj.(*databasev1.Database)
    if !ok {
        return nil, fmt.Errorf("expected a Database object for the newObj but got %T", newObj)
    }
    oldDB, ok := oldObj.(*databasev1.Database)
    if !ok {
        return nil, fmt.Errorf("expected a Database object for the oldObj but got %T", oldObj)
    }
    databaselog.Info("Validation for Database upon update", "name", database.GetName())

    // Prevent reducing storage size
    oldSize := parseStorageSize(oldDB.Spec.Storage.Size)
    newSize := parseStorageSize(database.Spec.Storage.Size)

    if newSize < oldSize {
        return nil, fmt.Errorf("cannot reduce storage from %s to %s", oldDB.Spec.Storage.Size, database.Spec.Storage.Size)
    }

    // Prevent changing database name
    if oldDB.Spec.DatabaseName != database.Spec.DatabaseName {
        return nil, fmt.Errorf("cannot change spec.databaseName from %s to %s", oldDB.Spec.DatabaseName, database.Spec.DatabaseName)
    }

    return nil, nil
}

// Helper function to parse storage size (e.g., "10Gi" -> 10)
func parseStorageSize(size string) int64 {
    if strings.HasSuffix(size, "Gi") {
        num := strings.TrimSuffix(size, "Gi")
        val, err := strconv.ParseInt(num, 10, 64)
        if err != nil {
            return 0
        }
        return val
    }
    return 0
}
```

### Задача 2.3: добавьте ValidateDelete (опционально)

```go
// ValidateDelete implements webhook.CustomValidator so a webhook will be registered for the type Database.
func (v *DatabaseCustomValidator) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    database, ok := obj.(*databasev1.Database)
    if !ok {
        return nil, fmt.Errorf("expected a Database object but got %T", obj)
    }
    databaselog.Info("Validation for Database upon deletion", "name", database.GetName())

    // Add any deletion validation logic
    // For example, prevent deletion if database has important data

    return nil, nil
}
```

## Упражнение 3: генерация манифестов

### Задача 3.1: сгенерируйте манифесты вебхука

```bash
# Generate manifests
make manifests

# Check webhook configuration was generated
ls -la config/webhook/

# Examine webhook configuration
cat config/webhook/manifests.yaml
```

### Задача 3.2: проверьте конфигурацию вебхука

```bash
# Check the configuration
cat config/webhook/manifests.yaml | grep -A 20 "ValidatingWebhookConfiguration"
```

## Упражнение 4: тестирование валидирующего вебхука

### Понимание тестирования вебхуков

В отличие от логики контроллера, вебхуки нельзя легко протестировать через `make run`, потому что:
- Вебхукам нужны TLS-сертификаты
- API-серверу Kubernetes (внутри кластера) нужно достучаться до эндпоинта вебхука
- При локальном запуске API-сервер не может обратиться обратно к вашему localhost

**Два подхода для разработки:**

| Подход | Команда | Работают ли вебхуки? | Когда использовать |
|----------|---------|----------------|----------|
| Локальная разработка | `make install && make run` | ❌ Нет | Тестирование логики контроллера/согласования |
| Развёртывание в кластере | `make deploy` | ✅ Да | Тестирование валидации вебхуком |

> **Примечание:** если вы использовали скрипт курса `scripts/setup-kind-cluster.sh` для создания кластера, cert-manager уже установлен. Проверьте командой: `kubectl get pods -n cert-manager`

### Задача 4.1: убедитесь, что Cert-Manager установлен

Если cert-manager не установлен:

```bash
# Install cert-manager in your cluster
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=120s
kubectl wait --for=condition=Available deployment/cert-manager-webhook -n cert-manager --timeout=120s
kubectl wait --for=condition=Available deployment/cert-manager-cainjector -n cert-manager --timeout=120s
```

### Задача 4.2: разверните оператор в кластер

Поскольку вебхуки должны работать внутри кластера, нужно собрать и развернуть:

```bash
# Build the container image
make docker-build IMG=postgres-operator:latest

# Load image into kind cluster
kind load docker-image postgres-operator:latest --name k8s-operators-course
```

Перед развёртыванием нужно установить `imagePullPolicy: IfNotPresent`, чтобы Kubernetes использовал локально загруженный образ вместо попытки скачать его из Docker Hub:

```bash
# Edit config/manager/manager.yaml and add imagePullPolicy
# Find the container spec and add: imagePullPolicy: IfNotPresent
```

Или используйте эту команду, чтобы применить патч:

```bash
# Add imagePullPolicy to manager.yaml
sed -i.bak 's/image: controller:latest/image: controller:latest\n          imagePullPolicy: IfNotPresent/' config/manager/manager.yaml
```

Теперь разверните:

```bash
# Deploy operator with webhooks to cluster
make deploy IMG=postgres-operator:latest
```

> **Используете Podman вместо Docker?**
> 
> Makefile использует переменную `CONTAINER_TOOL` (по умолчанию `docker`). Podman добавляет к образам префикс `localhost/`, поэтому используйте:
> ```bash
> # Build with podman (note: image will be localhost/postgres-operator:latest)
> make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
> 
> # Load image into kind (save to tarball, then load)
> podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
> kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
> rm /tmp/postgres-operator.tar
> 
> # Deploy operator - use localhost/ prefix to match the loaded image
> make deploy IMG=localhost/postgres-operator:latest
> ```

> **Получаете `ErrImagePull` или `ImagePullBackOff`?**
> 
> Это означает, что Kubernetes пытается скачать образ из Docker Hub вместо использования локального.
> 
> 1. Убедитесь, что в `config/manager/manager.yaml` установлено `imagePullPolicy: IfNotPresent`:
>    ```yaml
>    containers:
>    - name: manager
>      image: controller:latest
>      imagePullPolicy: IfNotPresent  # Add this line
>    ```
> 
> 2. **Пользователи Podman:** проверьте фактическое имя образа, загруженного в kind:
>    ```bash
>    podman exec k8s-operators-course-control-plane crictl images | grep postgres
>    ```
>    Если показывается `localhost/postgres-operator`, используйте это имя при развёртывании:
>    ```bash
>    make deploy IMG=localhost/postgres-operator:latest
>    ```

> **Совет:** для повседневной разработки контроллера вы всё ещё можете использовать `make install && make run`. Разворачивайте в кластер только когда нужно протестировать поведение вебхуков.

### Задача 4.3: проверьте, что вебхук зарегистрирован

```bash
# Check webhook configuration was created
kubectl get validatingwebhookconfigurations

# Check operator pods are running
kubectl get pods -n postgres-operator-system

# Check logs if needed
kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager
```

### Задача 4.4: протестируйте корректный ресурс

```bash
# Create valid Database
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

# Should succeed
kubectl get database valid-db
```

### Задача 4.5: протестируйте некорректные ресурсы

```bash
# Test invalid image
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-image
spec:
  image: nginx:latest  # Not PostgreSQL
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Should fail with validation error

# Test invalid storage for replicas
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-storage
spec:
  image: postgres:14
  replicas: 10  # Too many replicas
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi  # Too small
EOF

# Should fail with validation error
```

### Задача 4.6: протестируйте валидацию обновления

```bash
# Create database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: update-test
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 20Gi
EOF

# Try to reduce storage
kubectl patch database update-test --type merge -p '{"spec":{"storage":{"size":"10Gi"}}}'

# Should fail with validation error

# Try to change database name
kubectl patch database update-test --type merge -p '{"spec":{"databaseName":"newdb"}}'

# Should fail with validation error
```

## Упражнение 5: улучшение сообщений об ошибках

### Задача 5.1: добавьте контекст к ошибкам

Улучшите сообщения об ошибках:

```go
func (v *DatabaseCustomValidator) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    database, ok := obj.(*databasev1.Database)
    if !ok {
        return nil, fmt.Errorf("expected a Database object but got %T", obj)
    }
    databaselog.Info("Validation for Database upon creation", "name", database.GetName())

    var errors []string

    // Validate image
    if !strings.Contains(database.Spec.Image, "postgres") {
        errors = append(errors, fmt.Sprintf("spec.image: must be a PostgreSQL image, got '%s'. Valid examples: postgres:14, postgres:13", database.Spec.Image))
    }

    // Validate storage
    if database.Spec.Replicas != nil && *database.Spec.Replicas > 5 {
        if database.Spec.Storage.Size == "10Gi" {
            errors = append(errors, fmt.Sprintf("spec.storage.size: when replicas > 5, storage must be >= 50Gi, got '%s'", database.Spec.Storage.Size))
        }
    }

    if len(errors) > 0 {
        return nil, fmt.Errorf("validation failed: %s", strings.Join(errors, "; "))
    }

    return nil, nil
}
```

**Пересоберите и загрузите новый образ, как описано в** [Задаче 4.2: Разверните оператор в кластер](#задача-42-разверните-оператор-в-кластер), и перезапустите развёртывание, чтобы оно подхватило новый образ — `kubectl rollout restart deploy -n postgres-operator-system   postgres-operator-controller-manager`.

Теперь проверьте на примере ниже:
```
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-image-storage
spec:
  image: nginx:latest  # Not PostgreSQL
  replicas: 10
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi # less storage for replicas
EOF

# Should fail and error message should show both the spec.Image and spec.Storage errors
```

## Очистка

```bash
# Delete test resources
kubectl delete databases --all

# Stop operator (Ctrl+C)
```

## Итоги лабораторной

В этой лабораторной вы:
- Сгенерировали каркас валидирующего вебхука с помощью kubebuilder
- Реализовали пользовательскую логику валидации
- Протестировали на корректных и некорректных ресурсах
- Улучшили сообщения об ошибках
- Протестировали валидацию обновления

## Ключевые уроки

1. Kubebuilder легко генерирует каркас вебхуков в `internal/webhook/v1/`
2. Использует структуру `DatabaseCustomValidator`, реализующую `webhook.CustomValidator`
3. Методы получают `context.Context` первым параметром
4. `ValidateUpdate` получает и старый, и новый объект как `runtime.Object`
5. Приводите по типу `runtime.Object` к фактическому типу вашего ресурса
6. Предоставляйте понятные, применимые сообщения об ошибках
7. Тестируйте на корректных и некорректных ресурсах
8. Вебхуки запускаются после валидации схемы CRD
9. Сообщения об ошибках помогают пользователям исправлять проблемы

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Validating Webhook](../solutions/validating-webhook.go) — полная реализация валидирующего вебхука с пользовательской логикой валидации

## Дальнейшие шаги

Теперь давайте создадим мутирующий вебхук для установки значений по умолчанию!

**Навигация:** [← Предыдущая лабораторная: Контроль допуска](lab-01-admission-control.md) | [Связанный урок](../lessons/02-validating-webhooks.md) | [Следующая лабораторная: Мутирующие вебхуки →](lab-03-mutating-webhooks.md)
