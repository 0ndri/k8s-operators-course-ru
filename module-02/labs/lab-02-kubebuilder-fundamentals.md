---
layout: default
title: "Lab 02.2: Kubebuilder Fundamentals"
nav_order: 12
parent: "Модуль 2: Введение в операторы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 2.2: CLI Kubebuilder и структура проекта

**Связанный урок:** [Урок 2.2: Основы Kubebuilder](../lessons/02-kubebuilder-fundamentals.md)  
**Навигация:** [← Предыдущая лабораторная: Паттерн оператора](lab-01-operator-pattern.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Среда разработки →](lab-03-dev-environment.md)

## Цели

- Установить и проверить kubebuilder
- Разобраться в командах CLI kubebuilder
- Изучить структуру проекта kubebuilder
- Понять кодогенерацию

## Предварительные требования

- Установленный Go 1.21+
- Понимание операторов из [Урока 2.1](../lessons/01-operator-pattern.md)

## Упражнение 1: установка Kubebuilder

### Задача 1.1: проверьте, установлен ли Kubebuilder

```bash
# Check kubebuilder version
kubebuilder version
```

Если не установлен, переходите к установке.

### Задача 1.2: установите Kubebuilder

```bash
# Download kubebuilder
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/

# Verify installation
kubebuilder version
```

### Задача 1.3: проверьте установку

```bash
# Check kubebuilder is in PATH
which kubebuilder

# Check version
kubebuilder version

# List available commands
kubebuilder --help
```

## Упражнение 2: изучение команд Kubebuilder

### Задача 2.1: инициализируйте тестовый проект

```bash
# Create a test directory
mkdir -p /tmp/kubebuilder-test
cd /tmp/kubebuilder-test

# Initialize project
kubebuilder init --domain example.com --repo github.com/example/test-operator
```

**Обратите внимание:**
- Какие файлы были созданы?
- Какие каталоги были созданы?
- Что находится в Makefile?

### Задача 2.2: изучите структуру проекта

```bash
# List all files
find . -type f | head -20

# Examine main.go
cat ./cmd/main.go

# Examine Makefile
cat Makefile | head -30

# Check go.mod
cat go.mod
```

**Вопросы:**
1. Каково назначение `main.go`?
2. Какие цели (targets) Makefile доступны?
3. Какие зависимости есть в `go.mod`?

### Задача 2.3: создайте API

```bash
# Create an API
kubebuilder create api --group test --version v1 --kind TestResource
```

При запросах:
- Create Resource [y/n]: **y**
- Create Controller [y/n]: **y**

**Обратите внимание:**
- Какие новые файлы были созданы?
- Какие каталоги были добавлены?
- Что находится в каталоге API?

## Упражнение 3: понимание сгенерированного кода

### Задача 3.1: изучите типы API

```bash
# Look at generated types
cat api/v1/testresource_types.go
```

**Ключевые наблюдения:**
- Структура Spec
- Структура Status
- Маркеры kubebuilder (комментарии, начинающиеся с `// +kubebuilder:`)

### Задача 3.2: изучите контроллер

```bash
# Look at generated controller
cat internal/controller/testresource_controller.go
```

**Ключевые наблюдения:**
- Каркас функции Reconcile
- Маркеры RBAC
- Функция SetupWithManager

### Задача 3.3: сгенерируйте код

```bash
# Generate code
make generate

# Generate manifests
make manifests
```

**Обратите внимание:**
- Какие файлы были сгенерированы?
- Проверьте каталог `config/crd/bases/`
- Проверьте каталог `config/rbac/`

## Упражнение 4: изучение сгенерированных манифестов

### Задача 4.1: изучите CRD

```bash
# List generated CRDs
ls -la config/crd/bases/

# Examine CRD YAML
cat config/crd/bases/test.example.com_testresources.yaml | head -50
```

**Вопросы:**
1. Какая группа API используется?
2. Как называется ресурс?
3. Какая валидация включена?

### Задача 4.2: изучите RBAC

```bash
# List RBAC files
ls -la config/rbac/

# Examine role
cat config/rbac/role.yaml
```

**Вопросы:**
1. Какие разрешения предоставлены?
2. Как определяются разрешения?

## Упражнение 5: понимание процесса кодогенерации

### Задача 5.1: измените типы

Отредактируйте `api/v1/testresource_types.go` и добавьте поле:

```go
// TestResourceSpec defines the desired state of TestResource
type TestResourceSpec struct {
    // Message is a test message
    Message string `json:"message,omitempty"`
}
```

### Задача 5.2: перегенерируйте

```bash
# Regenerate code
make generate
make manifests

# Check CRD was updated
cat config/crd/bases/test.example.com_testresources.yaml | grep -A 5 message
```

**Наблюдение:** схема CRD была обновлена автоматически!

## Упражнение 6: изучение целей Makefile

### Задача 6.1: перечислите доступные цели

```bash
# List all Makefile targets
make help
```

### Задача 6.2: разберитесь в ключевых целях

**Важные цели:**
- `make generate` — генерирует код
- `make manifests` — генерирует манифесты
- `make install` — устанавливает CRD
- `make run` — запускает оператор локально
- `make docker-build` — собирает образ контейнера

### Задача 6.3: попробуйте некоторые цели

```bash
# Generate everything
make generate manifests

# Check what was created
ls -la config/crd/bases/
ls -la config/rbac/
```

## Упражнение 7: глубокое погружение в структуру проекта

### Задача 7.1: составьте карту структуры

Составьте мысленную карту проекта:

```
project-root/
├── api/                      # API type definitions
│   └── v1/                   # API version
├── internal/controller/      # Controller implementations
├── config/                   # Generated manifests
│   ├── crd/                  # CRD definitions
│   ├── rbac/                 # RBAC rules
│   └── manager/              # Manager deployment
├── main.go                   # Entry point
├── Makefile                  # Build targets
└── go.mod                    # Go dependencies
```

### Задача 7.2: разберитесь в каждом компоненте

Для каждого каталога поймите его назначение:

- **api/**: определения типов вашего пользовательского ресурса
- **internal/controller/**: ваша логика согласования
- **config/crd/**: сгенерированные YAML-файлы CRD
- **config/rbac/**: сгенерированные манифесты RBAC
- **main.go**: настраивает и запускает менеджер

## Очистка

```bash
# Remove test project
cd ~
rm -rf /tmp/kubebuilder-test
```

## Итоги лабораторной

В этой лабораторной вы:
- Установили и проверили kubebuilder
- Изучили команды CLI kubebuilder
- Создали тестовый проект
- Изучили структуру сгенерированного кода
- Разобрались в процессе кодогенерации
- Изучили структуру проекта

## Ключевые уроки

1. Kubebuilder предоставляет каркас для проектов операторов
2. `kubebuilder init` создаёт структуру проекта
3. `kubebuilder create api` генерирует типы API и контроллер
4. `make generate` и `make manifests` генерируют код и YAML
5. Структура проекта стандартизирована и организована
6. Маркеры kubebuilder управляют кодогенерацией

## Дальнейшие шаги

Теперь, когда вы понимаете kubebuilder, давайте настроим вашу полноценную среду разработки!

**Навигация:** [← Предыдущая лабораторная: Паттерн оператора](lab-01-operator-pattern.md) | [Связанный урок](../lessons/02-kubebuilder-fundamentals.md) | [Следующая лабораторная: Среда разработки →](lab-03-dev-environment.md)
