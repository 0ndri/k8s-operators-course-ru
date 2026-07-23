# Решения Модуля 8

Этот каталог содержит полные рабочие решения для лабораторных Модуля 8.

## Файлы

### Лабораторная 8.1 — мультиарендный оператор
- [**clusterdatabase-types.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/clusterdatabase-types.go): определения типов API ClusterDatabase (область действия на кластер)
- [**clusterdatabase-controller.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/clusterdatabase-controller.go): реализация контроллера ClusterDatabase
- [**multi-tenant-controller.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/multi-tenant-controller.go): паттерны мультиарендности и вспомогательные функции

### Лабораторная 8.2 — композиция операторов
- [**backup_types.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/backup_types.go): определения типов API Backup
- [**backup-operator.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/backup-operator.go): полный контроллер резервного копирования
- [**operator-coordination.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/operator-coordination.go): примеры координации операторов

### Лабораторная 8.3 — управление stateful-приложениями
- [**restore_types.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/restore_types.go): определения типов API Restore
- [**restore-controller.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/restore-controller.go): полная реализация контроллера Restore
- [**backup.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/backup.go): реализация функциональности резервного копирования
- [**restore.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/restore.go): реализация функциональности восстановления
- [**rolling-update.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/rolling-update.go): обработка скользящих обновлений
- [**Dockerfile**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-08/solutions/Dockerfile): Dockerfile с клиентскими инструментами PostgreSQL

## Использование

Эти решения можно использовать как:
- Справочный материал при создании продвинутых операторов
- Примеры паттернов мультиарендности
- Паттерны композиции операторов
- Примеры управления stateful-приложениями

## Интеграция

### Для мультиарендности (Лабораторная 8.1)

Используйте kubebuilder для генерации каркаса API ClusterDatabase, затем обращайтесь к решениям:

```bash
# 1. Scaffold the API
kubebuilder create api --group database --version v1 --kind ClusterDatabase

# 2. Reference clusterdatabase-types.go for type definitions
# 3. Reference clusterdatabase-controller.go for controller logic
# 4. Reference multi-tenant-controller.go for advanced patterns
```

Демонстрируемые ключевые концепции:
- Маркер `+kubebuilder:resource:scope=Cluster` для ресурсов области действия на кластер
- Поле `targetNamespace` для указания, где создавать ресурсы
- Владение на основе меток (поскольку OwnerReferences не могут пересекать границы областей действия)
- Финализаторы для очистки
- Проверка квот для каждого пространства имён/арендатора

### Для композиции операторов (Лабораторная 8.2)

Используйте kubebuilder для генерации каркаса API Backup, затем обращайтесь к решениям:

```bash
# 1. Scaffold the API (same group as Database to avoid multi-group setup)
kubebuilder create api --group database --version v1 --kind Backup --resource --controller

# 2. Reference backup_types.go for type definitions
# 3. Reference backup-operator.go for controller logic
# 4. Reference operator-coordination.go for coordination patterns
```

Демонстрируемые ключевые концепции:
- Одна группа API (`database`) для связанных ресурсов — избегает сложности с несколькими группами
- Поле `DatabaseRef` ссылается на Database для резервного копирования
- Контроллер ждёт готовности Database перед резервным копированием
- Условия статуса (`BackupReady`) координируют состояние между операторами
- Запланированные резервные копии с использованием cron-выражений

### Для stateful-приложений (Лабораторная 8.3)

Используйте kubebuilder для генерации каркаса API Restore, затем обращайтесь к решениям:

```bash
# 1. Scaffold the Restore API (same group as Database and Backup)
kubebuilder create api --group database --version v1 --kind Restore --resource --controller

# 2. Reference restore_types.go for type definitions
# 3. Reference restore-controller.go for complete controller implementation
# 4. Create backup package: mkdir -p internal/backup && cp backup.go internal/backup/
# 5. Create restore package: mkdir -p internal/restore && cp restore.go internal/restore/
# 6. Update Dockerfile to include PostgreSQL client tools (see Dockerfile in solutions)
# 7. Reference rolling-update.go for Database controller enhancements
```

Демонстрируемые ключевые концепции:
- Резервное копирование использует `pg_dump` для создания SQL-резервных копий
- Восстановление использует `psql` для восстановления из резервных копий
- Контроллер Restore координируется и с Database, и с Backup
- Скользящие обновления ждут готовности всех реплик
- Проверки согласованности данных проверяют статус репликации

**Важно:** поскольку резервное копирование/восстановление использует `pg_dump` и `psql`, вы должны обновить Dockerfile, включив клиентские инструменты PostgreSQL. Distroless-образ по умолчанию не включает эти инструменты. Пример с использованием `debian:bookworm-slim` с установленным `postgresql-client` см. в решении `Dockerfile`.

## Сравнение: Database против ClusterDatabase

| Возможность | Database | ClusterDatabase |
|---------|----------|-----------------|
| Область действия | Namespaced | Cluster |
| Пространство имён | Неявное | Явное (`targetNamespace`) |
| OwnerReferences | Да | Нет (используются метки) |
| Очистка | Автоматическая (GC) | Ручная (финализаторы) |
| Сценарий использования | Ресурсы команды | Управление платформой |

## Примечания

- Это полные рабочие примеры
- Они демонстрируют продвинутые паттерны
- Готовы к использованию в продакшене
- Следуют лучшим практикам
- CRD генерируются kubebuilder с помощью `make manifests`
