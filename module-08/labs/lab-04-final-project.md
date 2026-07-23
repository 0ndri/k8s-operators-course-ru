---
layout: default
title: "Lab 08.4: Final Project"
nav_order: 14
parent: "Модуль 8: Продвинутые темы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 8.4: Финальный проект

**Связанный урок:** [Урок 8.4: Практические паттерны и лучшие практики](../lessons/04-real-world-patterns.md)  
**Навигация:** [← Предыдущая лабораторная: Stateful-приложения](lab-03-stateful-applications.md) | [Обзор модуля](../README.md) | [Обзор курса](../../README.md)

## Цели

Создать полноценный, готовый к продакшену оператор для stateful-приложения, демонстрирующий все концепции, изученные на протяжении курса. Этот финальный проект объединяет всё, что вы изучили из Модулей 1–8, в единый исчерпывающий оператор.

## Предварительные требования

- Завершение всех предыдущих модулей (Модули 1–7)
- Завершение [Лабораторной 8.1](lab-01-multi-tenancy.md), [Лабораторной 8.2](lab-02-operator-composition.md) и [Лабораторной 8.3](lab-03-stateful-applications.md)
- Понимание всех концепций операторов:
  - Архитектура Kubernetes и механизмы API ([Модуль 1](../../module-01/README.md))
  - Паттерн оператора и Kubebuilder ([Модуль 2](../../module-02/README.md))
  - Controller runtime и согласование ([Модуль 3](../../module-03/README.md))
  - Продвинутые паттерны согласования ([Модуль 4](../../module-04/README.md))
  - Вебхуки и контроль допуска ([Модуль 5](../../module-05/README.md))
  - Тестирование и отладка ([Модуль 6](../../module-06/README.md))
  - Подготовка к продакшену ([Модуль 7](../../module-07/README.md))
- Рабочая среда разработки с kubebuilder, Go, Docker/Podman и kind

## Требования к проекту

Ваш финальный оператор должен включать:

1. **Полные CRUD-операции**
   - Create, Read, Update, Delete
   - Корректная обработка ошибок
   - Идемпотентные операции

2. **Отчётность о статусе**
   - Подресурс status
   - Условия (conditions)
   - Отслеживание прогресса
   - Наблюдаемое поколение (observed generation)

3. **Вебхуки**
   - Валидирующие вебхуки
   - Мутирующие вебхуки
   - Значения по умолчанию
   - Правила валидации

4. **Тестирование**
   - Модульные тесты
   - Интеграционные тесты
   - Покрытие тестами > 80%

5. **Продакшен-возможности**
   - Конфигурация RBAC
   - Усиление безопасности
   - Высокая доступность
   - Оптимизация производительности

6. **Продвинутые возможности**
   - Поддержка мультиарендности
   - Резервное копирование/восстановление
   - Скользящие обновления
   - Документация

## Структура проекта

```text
final-operator/
├── api/
│   └── v1/
│       ├── groupversion_info.go
│       └── <resource>_types.go
├── internal/controller/
│   └── <resource>_controller.go
├── config/
│   ├── crd/
│   ├── rbac/
│   ├── manager/
│   └── webhook/
├── internal/
│   ├── backup/
│   └── restore/
├── Dockerfile
├── Makefile
├── go.mod
├── go.sum
└── README.md
```

## Упражнение 1: выберите ваше приложение

### Варианты

1. **Оператор базы данных** (PostgreSQL, MySQL, MongoDB)
2. **Оператор очереди сообщений** (RabbitMQ, Kafka)
3. **Оператор кеша** (Redis, Memcached)
4. **Оператор поисковой системы** (Elasticsearch)
5. **Ваш выбор** (любое stateful-приложение)

### Задача 1.1: сгенерируйте каркас проекта оператора

Начните с создания нового проекта оператора с помощью kubebuilder:

```bash
# Create a new directory for your final project
mkdir -p ~/final-operator
cd ~/final-operator

# Initialize kubebuilder project
kubebuilder init --domain example.com --project-name final-operator

# Create your API (replace <resource> with your choice, e.g., Database, Cache, Queue)
kubebuilder create api \
  --group apps \
  --version v1 \
  --kind <Resource> \
  --resource --controller

# When prompted:
# Create Resource [y/n]: y
# Create Controller [y/n]: y
```

### Задача 1.2: определите исчерпывающие типы API

Отредактируйте `api/v1/<resource>_types.go`, чтобы создать готовый к продакшену API. Вот полный пример для оператора базы данных:

```go
package v1

import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DatabaseSpec defines the desired state of Database
type DatabaseSpec struct {
    // Image is the database image to use
    // +kubebuilder:validation:Required
    // +kubebuilder:default="postgres:14"
    Image string `json:"image"`

    // Replicas is the number of database replicas
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas *int32 `json:"replicas,omitempty"`

    // Storage is the storage configuration
    Storage StorageSpec `json:"storage"`

    // Resources are the resource requirements
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`

    // DatabaseName is the name of the database to create
    // +kubebuilder:validation:Required
    DatabaseName string `json:"databaseName"`

    // Username is the database user
    // +kubebuilder:validation:Required
    Username string `json:"username"`

    // PasswordSecretRef references a Secret containing the password
    // +optional
    PasswordSecretRef *corev1.SecretKeySelector `json:"passwordSecretRef,omitempty"`

    // Backup configuration
    // +optional
    Backup *BackupConfig `json:"backup,omitempty"`

    // Monitoring configuration
    // +optional
    Monitoring *MonitoringConfig `json:"monitoring,omitempty"`
}

// StorageSpec defines storage configuration
type StorageSpec struct {
    // Size is the storage size
    // +kubebuilder:validation:Required
    Size string `json:"size"`

    // StorageClassName is the storage class to use
    // +optional
    StorageClassName *string `json:"storageClassName,omitempty"`
}

// BackupConfig defines backup configuration
type BackupConfig struct {
    // Enabled enables automatic backups
    Enabled bool `json:"enabled"`

    // Schedule is the cron schedule for backups
    // +optional
    Schedule string `json:"schedule,omitempty"`

    // Retention is the number of backups to retain
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:default=7
    Retention int `json:"retention,omitempty"`
}

// MonitoringConfig defines monitoring configuration
type MonitoringConfig struct {
    // Enabled enables monitoring
    Enabled bool `json:"enabled"`

    // ServiceMonitor creates a ServiceMonitor for Prometheus
    // +optional
    ServiceMonitor bool `json:"serviceMonitor,omitempty"`
}

// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
    // Conditions represent the latest observations
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // Phase is the current phase
    // +kubebuilder:validation:Enum=Pending;Creating;Ready;Failed;Updating
    Phase string `json:"phase,omitempty"`

    // Ready indicates if the database is ready
    Ready bool `json:"ready,omitempty"`

    // ReadyReplicas is the number of ready replicas
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`

    // Endpoint is the database endpoint
    Endpoint string `json:"endpoint,omitempty"`

    // ObservedGeneration is the generation observed by the controller
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // LastBackupTime is when the last backup was completed
    // +optional
    LastBackupTime *metav1.Time `json:"lastBackupTime,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Ready",type="boolean",JSONPath=".status.ready"
// +kubebuilder:printcolumn:name="Replicas",type="integer",JSONPath=".status.readyReplicas"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// Database is the Schema for the databases API
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// DatabaseList contains a list of Database
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Database `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Database{}, &DatabaseList{})
}
```

### Задача 1.3: сгенерируйте CRD и проверьте

```bash
# Generate code and CRD manifests
make generate
make manifests

# Review the generated CRD
cat config/crd/bases/apps.example.com_databases.yaml

# Install CRDs
make install

# Verify CRD is installed
kubectl get crd databases.apps.example.com
```

## Упражнение 2: реализация основной функциональности

### Задача 2.1: реализуйте полную логику согласования

Отредактируйте `internal/controller/<resource>_controller.go`, чтобы реализовать полное согласование. Для полных примеров используйте эталонные реализации из [решений Модуля 3](../../module-03/solutions/database-controller.go) и [решений Модуля 4](../../module-04/solutions/state-machine-controller.go).

Ключевые компоненты для реализации:

- `Reconcile()` — основной цикл согласования с обработкой фаз
- `handlePending()` — обработка начального состояния
- `handleCreating()` — логика создания ресурсов
- `handleReady()` — мониторинг состояния Ready
- `handleUpdating()` — обработка обновлений
- `handleDeletion()` — очистка с финализаторами
- `reconcileSecret()` — управление Secret
- `reconcileStatefulSet()` — создание/обновление StatefulSet
- `reconcileService()` — управление Service
- `updateStatus()` — обновления статуса и условий

### Задача 2.2: реализуйте вспомогательные функции управления статусом

Добавьте исчерпывающие функции управления статусом. Для полного управления условиями используйте эталонные реализации из [решений Модуля 4](../../module-04/solutions/conditions-helpers.go).

## Упражнение 3: добавление вебхуков

### Задача 3.1: сгенерируйте каркас вебхуков

```bash
# Create validating and mutating webhooks
kubebuilder create webhook \
  --group apps \
  --version v1 \
  --kind Database \
  --programmatic-validation \
  --defaulting
```

### Задача 3.2: реализуйте валидирующий вебхук

Отредактируйте `internal/webhook/v1/database_webhook.go`, чтобы добавить исчерпывающую валидацию. Подробные примеры см. в [Лабораторной 2 Модуля 5](../../module-05/labs/lab-02-validating-webhooks.md).

Ключевые валидации для реализации:

- Валидация формата имени базы данных
- Валидация формата имени пользователя
- Валидация формата размера хранилища
- Валидация диапазона реплик
- Предотвращение изменения неизменяемых полей при обновлении

### Задача 3.3: реализуйте мутирующий вебхук

Добавьте логику установки значений по умолчанию для:

- Образа по умолчанию, если не указан
- Реплик по умолчанию, если не указаны
- Периода хранения резервных копий по умолчанию
- Лимитов ресурсов по умолчанию

### Задача 3.4: сгенерируйте манифесты вебхуков

```bash
# Generate webhook manifests
make manifests

# Review webhook configuration
cat config/webhook/manifests.yaml

# For local development, use cert-manager or manual certificates
# See Module 5 Lab 4 for webhook deployment details
```

## Упражнение 4: добавление тестирования

### Задача 4.1: напишите исчерпывающие модульные тесты

Отредактируйте `internal/controller/<resource>_controller_test.go`, чтобы добавить исчерпывающие тесты. Полные примеры см. в [решениях Модуля 6](../../module-06/solutions/database_controller_test.go).

Сценарии тестирования для покрытия:

- Создание ресурса
- Обновления ресурса
- Удаление ресурса
- Обновления статуса
- Обработка ошибок
- Обработка финализаторов

### Задача 4.2: напишите интеграционные тесты

Создайте `internal/controller/integration_test.go` для сквозных тестов. См. [Лабораторную 3 Модуля 6](../../module-06/labs/lab-03-integration-testing.md).

### Задача 4.3: запустите тесты и проверьте покрытие

```bash
# Setup envtest binaries
make setup-envtest

# Run all tests
make test

# Run with coverage
go test -coverprofile=coverage.out ./internal/controller/...
go tool cover -html=coverage.out

# Verify coverage is > 80%
go test -cover ./internal/controller/...
```

### Задача 4.4: добавьте примеры тестов для продвинутых возможностей

Добавьте тесты для:

- Функциональности резервного копирования/восстановления
- Сценариев мультиарендности
- Обработки ошибок и повторов
- Валидации вебхуками
- Обновлений условий статуса

Полные примеры тестов см. в [решениях Модуля 6](../../module-06/solutions/).

## Упражнение 5: продакшен-возможности

### Задача 5.1: настройте RBAC

```bash
# Generate RBAC manifests
make manifests

# Review RBAC configuration
cat config/rbac/role.yaml

# Optimize RBAC - remove unnecessary permissions
# Only grant permissions your operator actually needs
```

Проверьте и оптимизируйте сгенерированный RBAC. Лучшие практики см. в [Лабораторной 2 Модуля 7](../../module-07/labs/lab-02-rbac-security.md).

### Задача 5.2: усиление безопасности

#### Обновите Dockerfile для безопасности

Убедитесь, что ваш `Dockerfile` использует distroless-образы и запускается не от root. Полный пример см. в [решениях Модуля 7](../../module-07/solutions/Dockerfile).

#### Добавьте контексты безопасности

Обновите `config/manager/manager.yaml`, включив контексты безопасности. Полную конфигурацию безопасности см. в [решениях Модуля 7](../../module-07/solutions/security.yaml).

### Задача 5.3: включите высокую доступность

#### Включите выбор лидера

Обновите `cmd/main.go`, чтобы включить выбор лидера. Полную настройку HA см. в [Лабораторной 3 Модуля 7](../../module-07/labs/lab-03-high-availability.md).

#### Добавьте бюджет прерывания подов

Создайте `config/manager/pdb.yaml` для конфигурации бюджета прерывания подов.

### Задача 5.4: оптимизация производительности

#### Добавьте ограничение частоты

Обновите настройку контроллера, чтобы использовать ограничение частоты. Продвинутое ограничение частоты см. в [решениях Модуля 7](../../module-07/solutions/ratelimiter.go).

### Задача 5.5: добавьте наблюдаемость

Добавьте метрики и логирование. Полную настройку наблюдаемости см. в [Лабораторной 4 Модуля 6](../../module-06/labs/lab-04-debugging-observability.md).

## Упражнение 6: документация

### Задача 6.1: создайте исчерпывающий README

Создайте `README.md` в корне проекта с:

- Руководством по быстрому старту
- Обзором архитектуры
- Документацией API
- Примерами
- Руководством по устранению неполадок

### Задача 6.2: создайте примеры ресурсов

Создайте `config/samples/apps_v1_database.yaml` и дополнительные примеры в каталоге `examples/`:

- Пример базового использования
- Продвинутые сценарии
- Настройка мультиарендности
- Примеры резервного копирования/восстановления

### Задача 6.3: добавьте документацию API

Задокументируйте все поля API с понятными описаниями и примерами. Используйте маркеры kubebuilder для автоматической генерации документации.

## Упражнение 7: сборка и развёртывание

### Задача 7.1: соберите образ контейнера

```bash
# Build the image
make docker-build IMG=final-operator:v1.0.0

# For kind, load image
kind load docker-image final-operator:v1.0.0 --name k8s-operators-course

# Or push to registry
docker push final-operator:v1.0.0
```

### Задача 7.2: разверните оператор

```bash
# Update image in config/manager/manager.yaml
# Then deploy
make deploy IMG=final-operator:v1.0.0

# Verify deployment
kubectl get pods -n final-operator-system

# Check logs
kubectl logs -n final-operator-system -l control-plane=controller-manager
```

### Задача 7.3: протестируйте ваш оператор

```bash
# Create a test resource
kubectl apply -f config/samples/apps_v1_database.yaml

# Watch the resource
kubectl get database database-sample -w

# Verify resources were created
kubectl get statefulset,service,secret -l app=database-sample

# Test updates
kubectl patch database database-sample --type=merge -p '{"spec":{"replicas":3}}'

# Test deletion
kubectl delete database database-sample
```

## Чек-лист для сдачи

Используйте этот чек-лист, чтобы убедиться в полноте вашего оператора:

### Основная функциональность

- [ ] Реализованы полные CRUD-операции (Create, Read, Update, Delete)
- [ ] Корректная обработка ошибок повсюду
- [ ] Идемпотентная логика согласования
- [ ] Реализованы финализаторы для очистки

### Управление статусом

- [ ] Настроен подресурс status
- [ ] Условия реализованы и корректно обновляются
- [ ] Отслеживание фаз (Pending, Creating, Ready, Failed и т. д.)
- [ ] Отслеживание наблюдаемого поколения
- [ ] Отслеживание прогресса для длительных операций

### Вебхуки

- [ ] Реализован валидирующий вебхук
- [ ] Реализован мутирующий вебхук
- [ ] Значения по умолчанию установлены корректно
- [ ] Правила валидации исчерпывающие
- [ ] Настроены сертификаты вебхука

### Тестирование

- [ ] Написаны модульные тесты (покрытие >80%)
- [ ] Реализованы интеграционные тесты
- [ ] Набор тестов успешно выполняется (`make test`)
- [ ] Покрыты граничные случаи
- [ ] Протестированы сценарии ошибок

### Продакшен-возможности

- [ ] RBAC настроен и оптимизирован
- [ ] Безопасность усилена (distroless, не от root, контексты безопасности)
- [ ] Включена высокая доступность (выбор лидера)
- [ ] Производительность оптимизирована (ограничение частоты, конкурентность)
- [ ] Добавлена наблюдаемость (метрики, логирование)

### Продвинутые возможности

- [ ] Поддержка мультиарендности (если применимо)
- [ ] Функциональность резервного копирования/восстановления (если применимо)
- [ ] Скользящие обновления обрабатываются корректно
- [ ] Соблюдаются квоты ресурсов

### Документация

- [ ] README.md полон и включает:
  - Руководство по быстрому старту
  - Обзор архитектуры
  - Документацию API
  - Примеры
  - Руководство по устранению неполадок
- [ ] Предоставлены примеры ресурсов
- [ ] Комментарии в коде исчерпывающие
- [ ] Поля API задокументированы

### Упаковка

- [ ] Образ контейнера успешно собирается
- [ ] Образ опубликован в реестр (или загружен в kind)
- [ ] Оператор успешно развёртывается
- [ ] Все ресурсы создаются корректно
- [ ] Оператор работает от начала до конца

## Очистка

После завершения проекта:

```bash
# Delete test resources
kubectl delete database --all --all-namespaces

# Undeploy operator
make undeploy

# Uninstall CRDs
make uninstall

# Clean up kind cluster (if using)
kind delete cluster --name k8s-operators-course
```

## Чек-лист для самооценки

Используйте этот чек-лист, чтобы убедиться, что ваш оператор соответствует готовым к продакшену стандартам:

### Функциональность

- **Основные операции**: все CRUD-операции работают корректно
- **Граничные случаи**: аккуратно обрабатывает граничные случаи (удаление, обновления, сбои)
- **Обработка ошибок**: корректная обработка ошибок и логика повторов
- **Управление статусом**: статус точно отражает состояние ресурса
- **Вебхуки**: валидация и мутация работают корректно

### Качество кода

- **Структура**: хорошо организован, следует лучшим практикам Go
- **Читаемость**: понятные имена переменных, комментарии где нужно
- **Идемпотентность**: согласование идемпотентно
- **Управление ресурсами**: правильное использование финализаторов, ссылок-владельцев
- **Сообщения об ошибках**: понятные, применимые сообщения об ошибках

### Стандарты тестирования

- **Покрытие**: покрытие тестами > 80%
- **Модульные тесты**: исчерпывающие модульные тесты для логики контроллера
- **Интеграционные тесты**: сквозные интеграционные тесты
- **Граничные случаи**: тесты покрывают сценарии ошибок и граничные случаи
- **Качество тестов**: тесты сопровождаемы и хорошо структурированы

### Готовность к продакшену

- **Безопасность**: использует distroless-образы, не от root, контексты безопасности
- **RBAC**: минимальная конфигурация RBAC по принципу наименьших привилегий
- **Высокая доступность**: включён выбор лидера
- **Производительность**: настроены ограничение частоты, лимиты конкурентности
- **Наблюдаемость**: реализованы метрики и логирование

### Стандарты документации

- **README**: исчерпывающий README с быстрым стартом
- **Документация API**: все поля API задокументированы
- **Примеры**: предоставлено несколько примеров ресурсов
- **Устранение неполадок**: задокументированы распространённые проблемы и решения
- **Комментарии в коде**: важная логика объяснена

## Интеграция с предыдущими модулями

Этот финальный проект объединяет концепции из всех предыдущих модулей:

- **Модуль 1**: понимание архитектуры Kubernetes и механизмов API
- **Модуль 2**: использование Kubebuilder для генерации каркаса операторов
- **Модуль 3**: controller runtime и паттерны согласования
- **Модуль 4**: продвинутые паттерны (условия, финализаторы, отслеживание)
- **Модуль 5**: вебхуки и контроль допуска
- **Модуль 6**: тестирование и наблюдаемость
- **Модуль 7**: подготовка к продакшену (упаковка, безопасность, HA)
- **Модуль 8**: продвинутые темы (мультиарендность, композиция, stateful-приложения)

Эталонные решения из предыдущих модулей:

- [Решения Модуля 3](../../module-03/solutions/) — реализация контроллера
- [Решения Модуля 4](../../module-04/solutions/) — продвинутые паттерны
- [Решения Модуля 5](../../module-05/solutions/) — вебхуки
- [Решения Модуля 6](../../module-06/solutions/) — тестирование
- [Решения Модуля 7](../../module-07/solutions/) — продакшен-возможности
- [Решения Модуля 8](../solutions/) — продвинутые возможности

## Решения

Полные примеры решений и эталонные реализации доступны:

- **Решения предыдущих модулей**: используйте эталонные решения из Модулей 3–7 для паттернов реализации
- **Решения Модуля 8**: см. [каталог решений](../solutions/) для:
  - Паттернов мультиарендного оператора
  - Примеров композиции операторов
  - Реализаций резервного копирования/восстановления
  - Паттернов скользящих обновлений

## Итоги лабораторной

В этой лабораторной с финальным проектом вы:

1. **Спроектировали полноценный API** — создали исчерпывающий CRD со spec и status
2. **Реализовали полное согласование** — построили полноценный контроллер со всеми CRUD-операциями
3. **Добавили вебхуки** — реализовали валидирующие и мутирующие вебхуки
4. **Написали исчерпывающие тесты** — создали модульные и интеграционные тесты с покрытием >80%
5. **Настроили продакшен-возможности** — настроили RBAC, безопасность, HA и оптимизации производительности
6. **Создали документацию** — написали README, примеры и документацию API
7. **Собрали и развернули** — упаковали оператор как образ контейнера и развернули в кластер

## Ключевые уроки

Благодаря этому финальному проекту вы продемонстрировали владение:

1. **Жизненным циклом разработки операторов** — от генерации каркаса до продакшен-развёртывания
2. **Паттернами API Kubernetes** — CRD, подресурсы status, условия, финализаторы
3. **Паттернами контроллеров** — согласование, идемпотентность, обработка ошибок
4. **Контролем допуска** — валидирующие и мутирующие вебхуки
5. **Стратегиями тестирования** — модульные тесты с envtest, интеграционные тесты
6. **Готовностью к продакшену** — безопасность, HA, производительность, наблюдаемость
7. **Лучшими практиками** — организация кода, документация, примеры

## Дальнейшие шаги

После завершения этого курса вы можете:

1. **Создавать реальные операторы** — применять эти паттерны для создания операторов для ваших приложений
2. **Вносить вклад в open source** — участвовать в существующих операторах или создавать новые
3. **Продвинутые темы** — изучить:
   - Operator SDK (альтернатива Kubebuilder)
   - Operator Lifecycle Manager (OLM)
   - Operator Framework
   - Мультикластерные операторы
   - Метрики и дашборды операторов

## Поделитесь своим проектом

Мы будем рады увидеть, что вы создали! Если вы завершили финальный проект и хотите поделиться им:

1. **Опубликуйте в LinkedIn** — поделитесь своим проектом оператора, тем, что вы изучили, и тем, что создали
2. **Отметьте автора курса** — отметьте [Piyush Jajoo](https://www.linkedin.com/in/pjajoo) в своём посте
3. **Укажите детали** — поделитесь:
   - Какой оператор вы создали
   - Ключевые реализованные возможности
   - Что вы изучили из курса
   - Ссылку на ваш репозиторий GitHub (если публичный)

Я постараюсь в свободное время просмотреть ваш код и дать обратную связь!

## Дополнительные ресурсы

- [Kubebuilder Book](https://book.kubebuilder.io/) — исчерпывающая документация Kubebuilder
- [Соглашения об API Kubernetes](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — рекомендации по проектированию API
- [Лучшие практики операторов](https://sdk.operatorframework.io/docs/best-practices/) — лучшие практики Operator SDK
- [Примеры операторов](https://github.com/operator-framework/awesome-operators) — список примеров операторов

## Поздравляем! 🎉

Вы завершили весь курс **«Разработка операторов Kubernetes»**!

Теперь у вас есть:

- ✅ Глубокое понимание архитектуры Kubernetes и операторов
- ✅ Практический опыт создания готовых к продакшену операторов
- ✅ Знание лучших практик и паттернов
- ✅ Навыки создания операторов для любого приложения

**Теперь вы готовы создавать готовые к продакшену операторы Kubernetes!**

**Навигация:** [← Предыдущая лабораторная: Stateful-приложения](lab-03-stateful-applications.md) | [Связанный урок](../lessons/04-real-world-patterns.md) | [Обзор модуля](../README.md) | [Обзор курса](../../README.md)
