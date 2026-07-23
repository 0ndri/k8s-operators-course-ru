---
layout: default
title: "Lab 05.3: Mutating Webhooks"
nav_order: 13
parent: "Модуль 5: Вебхуки и контроль допуска"
grand_parent: Модули
mermaid: true
---

# Лабораторная 5.3: Создание мутирующего вебхука

**Связанный урок:** [Урок 5.3: Реализация мутирующих вебхуков](../lessons/03-mutating-webhooks.md)  
**Навигация:** [← Предыдущая лабораторная: Валидирующие вебхуки](lab-02-validating-webhooks.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Развёртывание вебхуков →](lab-04-webhook-deployment.md)

## Цели

- Добавить мутирующий вебхук к существующему валидирующему
- Реализовать логику установки значений по умолчанию
- Протестировать сценарии мутации
- Обеспечить идемпотентность

## Предварительные требования

- Завершение [Лабораторной 5.2](lab-02-validating-webhooks.md)
- Оператор Database с валидирующим вебхуком
- Понимание паттернов установки значений по умолчанию

## Упражнение 1: добавление мутирующего вебхука

Поскольку мы уже создали валидирующий вебхук в Лабораторной 5.2, наш файл вебхука уже существует в `internal/webhook/v1/database_webhook.go`. Мы добавим мутирующую логику (установку значений по умолчанию) в этот файл.

> **Примечание:** если бы вы начинали с нуля, вы бы запустили:
> ```bash
> kubebuilder create webhook --group database --version v1 --kind Database --defaulting
> ```
> Но поскольку у нас уже есть вебхук, мы добавим дефолтер вручную.

### Задача 1.1: разберитесь в интерфейсе CustomDefaulter

Новый паттерн kubebuilder использует интерфейс `webhook.CustomDefaulter`:

```go
type CustomDefaulter interface {
    Default(ctx context.Context, obj runtime.Object) error
}
```

### Задача 1.2: добавьте дефолтер в настройку вебхука

Отредактируйте `internal/webhook/v1/database_webhook.go`, чтобы обновить функцию настройки вебхука:

```go
// SetupDatabaseWebhookWithManager registers the webhook for Database in the manager.
func SetupDatabaseWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).For(&databasev1.Database{}).
        WithValidator(&DatabaseCustomValidator{}).
        WithDefaulter(&DatabaseCustomDefaulter{}).
        Complete()
}
```

### Задача 1.3: добавьте структуру дефолтера и маркер

Добавьте следующее в `internal/webhook/v1/database_webhook.go`:

```go
// +kubebuilder:webhook:path=/mutate-database-example-com-v1-database,mutating=true,failurePolicy=fail,sideEffects=None,groups=database.example.com,resources=databases,verbs=create;update,versions=v1,name=mdatabase-v1.kb.io,admissionReviewVersions=v1

// DatabaseCustomDefaulter struct is responsible for setting default values on the Database resource.
type DatabaseCustomDefaulter struct {
    // Add fields as needed for defaulting
}

var _ webhook.CustomDefaulter = &DatabaseCustomDefaulter{}
```

## Упражнение 2: реализация логики установки значений по умолчанию

### Задача 2.1: добавьте метод Default

Добавьте метод `Default` в `internal/webhook/v1/database_webhook.go`:

```go
// Default implements webhook.CustomDefaulter so a webhook will be registered for the type Database.
func (d *DatabaseCustomDefaulter) Default(ctx context.Context, obj runtime.Object) error {
    database, ok := obj.(*databasev1.Database)
    if !ok {
        return fmt.Errorf("expected a Database object but got %T", obj)
    }
    databaselog.Info("Defaulting for Database", "name", database.GetName())

    // Set default image if not specified
    if database.Spec.Image == "" {
        database.Spec.Image = "postgres:14"
    }

    // Set default replicas if not specified
    if database.Spec.Replicas == nil {
        replicas := int32(1)
        database.Spec.Replicas = &replicas
    }

    // Set default storage class if not specified
    if database.Spec.Storage.StorageClass == "" {
        database.Spec.Storage.StorageClass = "standard"
    }

    return nil
}
```

### Задача 2.2: добавьте значения по умолчанию с учётом контекста

Дополните метод Default значениями по умолчанию на основе пространства имён:

```go
func (d *DatabaseCustomDefaulter) Default(ctx context.Context, obj runtime.Object) error {
    database, ok := obj.(*databasev1.Database)
    if !ok {
        return fmt.Errorf("expected a Database object but got %T", obj)
    }
    databaselog.Info("Defaulting for Database", "name", database.GetName(), "namespace", database.GetNamespace())

    // Set defaults based on namespace
    if database.Namespace == "production" {
        // Production defaults - ensure minimum 3 replicas
        // Note: We check < 3 instead of nil because CRD schema defaults may already set replicas=1
        if database.Spec.Replicas == nil || *database.Spec.Replicas < 3 {
            replicas := int32(3)
            database.Spec.Replicas = &replicas
        }
    }
    // For non-production, CRD schema default of 1 replica is fine

    // Common defaults
    if database.Spec.Storage.StorageClass == "" {
        database.Spec.Storage.StorageClass = "standard"
    }

    // Add labels (idempotent)
    if database.Labels == nil {
        database.Labels = make(map[string]string)
    }
    if _, exists := database.Labels["managed-by"]; !exists {
        database.Labels["managed-by"] = "database-operator"
    }

    // Add annotations (idempotent)
    if database.Annotations == nil {
        database.Annotations = make(map[string]string)
    }
    if _, exists := database.Annotations["database.example.com/version"]; !exists {
        database.Annotations["database.example.com/version"] = "v1"
    }

    return nil
}
```

> **Примечание:** мы проверяем `< 3`, а не `nil`, для реплик, потому что значения по умолчанию схемы CRD (через `+kubebuilder:default=1`) применяются до запуска вебхуков. Это гарантирует, что продакшен-пространства всегда получают минимум 3 реплики.

## Упражнение 3: обеспечение идемпотентности

### Задача 3.1: разберитесь в идемпотентности

Мутации должны быть **идемпотентными** — их многократное применение должно давать один и тот же результат:

```go
// Idempotent: Only set if not already set
if database.Spec.Image == "" {
    database.Spec.Image = "postgres:14"
}
// If already set, doesn't change

// Idempotent: Check before adding to map
if _, exists := database.Labels["managed-by"]; !exists {
    database.Labels["managed-by"] = "database-operator"
}
// If already exists, doesn't add again
```

## Упражнение 4: развёртывание и тестирование мутирующего вебхука

### Задача 4.1: включите MutatingWebhookConfiguration в Kustomization

Поскольку мы добавили мутирующий вебхук вручную, нужно раскомментировать замены (replacements) MutatingWebhookConfiguration в `config/default/kustomization.yaml`, чтобы cert-manager мог внедрить CA bundle:

```bash
cd ~/postgres-operator

# Uncomment the MutatingWebhookConfiguration section (around lines 188-217)
# Find the section that says "Uncomment the following block if you have a DefaultingWebhook"
# and uncomment it.

# Verify it's uncommented - should show MutatingWebhookConfiguration without # prefix
grep -A 5 "DefaultingWebhook" config/default/kustomization.yaml
```

### Задача 4.2: сгенерируйте манифесты

```bash
# Generate manifests (includes new mutating webhook configuration)
make manifests

# Verify mutating webhook manifest was generated
grep "mutating" config/webhook/manifests.yaml
```

### Задача 4.3: снимите развёртывание и удалите устаревшие вебхуки

Поскольку мы добавили новый вебхук, нужно полностью переразвернуть. Также удалите любые устаревшие конфигурации вебхуков от предыдущих развёртываний:

```bash
# Remove existing deployment
make undeploy

# Clean up any stale webhook configurations (from previous deployments without proper prefixes)
kubectl delete validatingwebhookconfiguration validating-webhook-configuration 2>/dev/null || true
kubectl delete mutatingwebhookconfiguration mutating-webhook-configuration 2>/dev/null || true

# Verify cleanup
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
# Should only show cert-manager and ingress-nginx webhooks, not our old ones

# Wait for resources to be deleted
kubectl get all -n postgres-operator-system
# Should show "No resources found"
```

### Задача 4.4: пересоберите и разверните

```bash
# Rebuild the image
# For Docker:
make docker-build IMG=postgres-operator:latest

# For Podman:
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman

# Load image into kind
# For Docker:
kind load docker-image postgres-operator:latest --name k8s-operators-course

# For Podman:
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar

# Deploy
# For Docker:
make deploy IMG=postgres-operator:latest

# For Podman:
make deploy IMG=localhost/postgres-operator:latest
```

### Задача 4.5: дождитесь сертификатов

cert-manager нужно время, чтобы сгенерировать сертификаты и внедрить CA bundle:

```bash
# Wait for certificate to be ready
kubectl get certificate -n postgres-operator-system -w

# Wait for pod to be ready
kubectl wait --for=condition=Ready pod -l control-plane=controller-manager \
  -n postgres-operator-system --timeout=120s

# Check operator logs - should see both webhooks registered
kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager | grep -i webhook
```

### Задача 4.6: проверьте, что оба вебхука зарегистрированы

```bash
# Check both webhooks are configured
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations

# You should see both:
# - postgres-operator-validating-webhook-configuration
# - postgres-operator-mutating-webhook-configuration
```

### Задача 4.7: протестируйте минимальный ресурс

```bash
# Create resource with minimal spec (missing image, replicas)
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: minimal-db
spec:
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
  # image and replicas should be defaulted
EOF

# Check defaults were applied
echo "Image:"
kubectl get database minimal-db -o jsonpath='{.spec.image}'
echo
echo "Replicas:"
kubectl get database minimal-db -o jsonpath='{.spec.replicas}'
echo
echo "Storage Class:"
kubectl get database minimal-db -o jsonpath='{.spec.storage.storageClass}'
echo
echo "Managed-by label:"
kubectl get database minimal-db -o jsonpath='{.metadata.labels.managed-by}'
echo
```

### Задача 4.8: протестируйте значения по умолчанию на основе пространства имён

Наш вебхук проверяет `replicas < 3` (а не просто `nil`) для продакшен-пространства, поэтому он работает, даже когда значения по умолчанию схемы CRD уже установили `replicas=1`.

```bash
# Clean up previous test resources
kubectl delete database --all --ignore-not-found
kubectl delete database --all -n production --ignore-not-found

# Create in default namespace (should stay at 1 replica - CRD default)
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: dev-db
spec:
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Create in production namespace (should be bumped to 3 replicas)
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: prod-db
  namespace: production
spec:
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Check results
echo "=== Default namespace (should be 1 replica) ==="
kubectl get database dev-db -o jsonpath='Replicas: {.spec.replicas}'
echo

echo "=== Production namespace (should be 3 replicas) ==="
kubectl get database prod-db -n production -o jsonpath='Replicas: {.spec.replicas}'
echo
```

> **Ключевой вывод:** значения по умолчанию схемы CRD (`+kubebuilder:default`) применяются до вебхуков. Чтобы переопределить их, проверяйте значение по умолчанию (например, `< 3`), а не просто наличие `nil`.

## Упражнение 5: тестирование порядка мутаций

### Задача 5.1: проверьте, что мутация выполняется до валидации

```bash
# Create resource that would fail validation without defaults
# (missing image, but mutating webhook will set it to postgres:14)
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-order
spec:
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Should succeed because:
# 1. Mutating webhook sets image to postgres:14
# 2. Validating webhook validates it's a postgres image
kubectl get database test-order -o jsonpath='{.spec.image}'
echo
```

## Очистка

```bash
# Delete test resources
kubectl delete databases --all -A
kubectl delete namespace production --ignore-not-found
```

## Итоги лабораторной

В этой лабораторной вы:
- Добавили мутирующий вебхук к существующему валидирующему
- Реализовали логику установки значений по умолчанию с помощью интерфейса `CustomDefaulter`
- Добавили значения по умолчанию с учётом контекста
- Обеспечили идемпотентность
- Протестировали сценарии мутации
- Проверили порядок мутаций

## Ключевые уроки

1. Добавляйте мутирующий вебхук в существующий `internal/webhook/v1/database_webhook.go`
2. Используйте интерфейс `webhook.CustomDefaulter` с отдельной структурой
3. Метод `Default` получает `context.Context` и `runtime.Object`
4. Регистрируйте дефолтер через `.WithDefaulter(&DatabaseCustomDefaulter{})`
5. Значения по умолчанию могут учитывать контекст (пространство имён и т. д.)
6. Мутации должны быть идемпотентными
7. Мутирующие вебхуки запускаются до валидирующих

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Mutating Webhook](../solutions/mutating-webhook.go) — полная реализация мутирующего вебхука с логикой установки значений по умолчанию

## Дальнейшие шаги

Теперь давайте изучим управление сертификатами и развёртывание!

**Навигация:** [← Предыдущая лабораторная: Валидирующие вебхуки](lab-02-validating-webhooks.md) | [Связанный урок](../lessons/03-mutating-webhooks.md) | [Следующая лабораторная: Развёртывание вебхуков →](lab-04-webhook-deployment.md)
