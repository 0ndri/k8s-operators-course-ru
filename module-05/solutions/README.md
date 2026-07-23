# Решения Модуля 5

Этот каталог содержит полные рабочие решения для лабораторных Модуля 5.

## Файлы

- [**validating-webhook.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-05/solutions/validating-webhook.go): полная реализация валидирующего вебхука
- [**mutating-webhook.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-05/solutions/mutating-webhook.go): полная реализация мутирующего вебхука

## Использование

Эти решения можно использовать как:
- Справочный материал при реализации собственных вебхуков
- Отправную точку, если вы застряли
- Примеры лучших практик

## Интеграция

Чтобы использовать эти решения в вашем операторе:

1. Скопируйте код вебхука в `internal/webhook/v1/database_webhook.go`
2. Убедитесь, что ваши типы API соответствуют структуре
3. Запустите `make generate` и `make manifests`

## Тестирование вебхуков

Вебхукам нужны TLS-сертификаты, и они должны быть достижимы для API-сервера Kubernetes. В отличие от контроллеров, вебхуки нельзя легко протестировать через `make run`.

### Вариант 1: развёртывание в кластер (рекомендуется для тестирования вебхуков)

```bash
# Ensure cert-manager is installed (handles TLS certificates)
kubectl get pods -n cert-manager

# Build and load image into kind
make docker-build IMG=postgres-operator:latest
kind load docker-image postgres-operator:latest --name k8s-operators-course

# Deploy operator with webhooks
make deploy IMG=postgres-operator:latest
```

### Вариант 2: локальная разработка (только логика контроллера)

```bash
# For testing controller/reconciliation logic (webhooks won't be invoked)
make install && make run
```

### Пользователи Podman

```bash
# Build with podman
make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman

# Load into kind via tarball
podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
rm /tmp/postgres-operator.tar

# Deploy with localhost prefix
make deploy IMG=localhost/postgres-operator:latest
```

## Примечания

- Код вебхука размещается в каталоге `internal/webhook/v1/`
- Использует интерфейсы `webhook.CustomValidator` и `webhook.CustomDefaulter`
- Методы получают `context.Context` первым параметром
- `ValidateUpdate` получает и старый, и новый объект как `runtime.Object`
- Сообщения об ошибках понятны и применимы
- Мутации идемпотентны
- Валидация покрывает распространённые сценарии

## Важно: значения по умолчанию схемы CRD против вебхука

Значения по умолчанию схемы CRD (через маркеры `+kubebuilder:default`) применяются **до** запуска мутирующих вебхуков. Это означает:

- Если у вашего CRD есть `+kubebuilder:default=1` для реплик, `Spec.Replicas` будет равно `1` (а не `nil`), когда запускается ваш вебхук
- Чтобы переопределить значения по умолчанию CRD в вебхуках, проверяйте значение по умолчанию, а не `nil`

Пример:
```go
// Instead of checking nil (won't work if CRD has default):
if database.Spec.Replicas == nil {
    replicas := int32(3)
    database.Spec.Replicas = &replicas
}

// Check for the value you want to override:
if database.Spec.Replicas == nil || *database.Spec.Replicas < 3 {
    replicas := int32(3)
    database.Spec.Replicas = &replicas
}
```

**Лучшая практика:** используйте значения по умолчанию схемы CRD для простых статических значений, а вебхуки — для значений с учётом контекста (например, на основе пространства имён).
