---
layout: default
title: "Lab 04.3: Watching Indexing"
nav_order: 13
parent: "Модуль 4: Продвинутое согласование"
grand_parent: Модули
mermaid: true
---

# Лабораторная 4.3: Настройка отслеживания и индексов

**Связанный урок:** [Урок 4.3: Отслеживание и индексирование](../lessons/03-watching-indexing.md)  
**Навигация:** [← Предыдущая лабораторная: Финализаторы](lab-02-finalizers-cleanup.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Продвинутые паттерны →](lab-04-advanced-patterns.md)

## Цели

- Настроить отслеживание зависимых ресурсов
- Создать индексы для эффективного поиска
- Обрабатывать события отслеживания
- Протестировать поведение отслеживания

## Предварительные требования

- Завершение [Лабораторной 4.2](lab-02-finalizers-cleanup.md)
- Оператор Database с финализаторами
- Понимание паттернов отслеживания

## Упражнение 1: отслеживание подчинённых ресурсов

### Задача 1.1: обновите SetupWithManager

Мы уже изменили `SetupWithManager`, чтобы отслеживать подчинённые ресурсы:

```go
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).  // Watch owned StatefulSets
        Owns(&corev1.Service{}).      // Watch owned Services
        Owns(&corev1.Secret{}).       // Watch owned Secrets
        Complete(r)
}
```

### Задача 1.2: протестируйте поведение отслеживания

```bash
# Install and run operator
make install
make run

# Create Database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-db
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Manually delete StatefulSet
kubectl delete statefulset test-db

# Watch operator logs - should detect and recreate

# Validate the deleted statefulset appears
kubectl get statefulset test-db

# Delete the database
kubectl delete database test-db
```

## Упражнение 2: отслеживание неподчинённых ресурсов

### Задача 2.1: отслеживание Secret

Добавьте отслеживание Secret, на которые ссылаются Database:

```go
import (
    "sigs.k8s.io/controller-runtime/pkg/handler"
    "sigs.k8s.io/controller-runtime/pkg/source"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
    "k8s.io/apimachinery/pkg/types"
)

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        // deliberately removing Owns(&corev1.Secret{}). to demonstrate non-owned resources
        Watches(
			&corev1.Secret{},
			handler.EnqueueRequestsFromMapFunc(r.findDatabasesForSecret),
		).
        Complete(r)
}

func (r *DatabaseReconciler) findDatabasesForSecret(ctx context.Context, secret client.Object) []reconcile.Request {
	databases := &databasev1.DatabaseList{}
	r.List(context.Background(), databases)

	var requests []reconcile.Request
	for _, db := range databases.Items {
		// If Database references this Secret
		if r.secretName(&db) == secret.GetName() &&
			db.Namespace == secret.GetNamespace() {
			requests = append(requests, reconcile.Request{
				NamespacedName: types.NamespacedName{
					Name:      db.Name,
					Namespace: db.Namespace,
				},
			})
		}
	}
	return requests
}
```

### Задача 2.2: протестируйте отслеживание Secret

```bash
# Install and run operator
make install
make run

# Create Database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-db
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Update the Secret
kubectl patch secret test-db-credentials --type merge -p '{"data":{"password":"newpassword"}}'

# Watch operator logs - should reconcile Database
```

## Упражнение 3: создание индексов

Индексы позволяют эффективно искать ресурсы по значениям полей без сканирования всех объектов.

### Задача 3.1: настройте индекс

Добавьте индекс для поля `image`, чтобы быстро находить все Database, использующие конкретную версию PostgreSQL:

```go
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // Create index for image field
    if err := mgr.GetFieldIndexer().IndexField(
        context.Background(),
        &databasev1.Database{},
        "spec.image",
        func(obj client.Object) []string {
            db, ok := obj.(*databasev1.Database)
            if !ok {
                return nil
            }
            if db.Spec.Image != "" {
                return []string{db.Spec.Image}
            }
            return nil
        },
    ); err != nil {
        return err
    }
    
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Watches(
            &corev1.Secret{},
            handler.EnqueueRequestsFromMapFunc(r.findDatabasesForSecret),
        ).
        Complete(r)
}
```

### Задача 3.2: используйте индекс в запросе

Используйте индекс, чтобы эффективно находить все Database, использующие конкретный образ:

```go
// findDatabasesByImage finds all Databases using a specific PostgreSQL image
func (r *DatabaseReconciler) findDatabasesByImage(ctx context.Context, image string) ([]databasev1.Database, error) {
    databases := &databasev1.DatabaseList{}
    err := r.List(ctx, databases, client.MatchingFields{
        "spec.image": image,
    })
    
    if err != nil {
        return nil, err
    }
    
    return databases.Items, nil
}
```

### Задача 3.3: протестируйте использование индекса

```bash
# Install and run operator
make install
make run

# Create databases with different images
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: db-postgres14
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
---
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: db-postgres15
spec:
  image: postgres:15
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
---
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: db-postgres14-2
spec:
  image: postgres:14
  replicas: 1
  databaseName: testdb
  username: admin
  storage:
    size: 5Gi
EOF

# The index allows efficient lookup - finding all postgres:14 databases
# doesn't require scanning every Database object
```

> **Примечание:** индексы особенно полезны, когда у вас много ресурсов и нужно быстро находить подмножества. Без индекса `List()` с сопоставлением по полям пришлось бы сканировать все объекты.

## Упражнение 4: предикаты событий

### Задача 4.1: добавьте предикаты

Фильтруйте события, чтобы согласовывать только при важных изменениях.

> **Важно:** при фильтрации обновлений StatefulSet вы должны включать **как** изменения spec (Generation), ТАК И изменения status (ReadyReplicas). Иначе Database никогда не станет Ready, потому что обновления status будут отфильтрованы!

```go
import (
    "sigs.k8s.io/controller-runtime/pkg/builder"
    "sigs.k8s.io/controller-runtime/pkg/predicate"
    "sigs.k8s.io/controller-runtime/pkg/event"
)

func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}, builder.WithPredicates(predicate.Funcs{
            UpdateFunc: func(e event.UpdateEvent) bool {
                oldSS := e.ObjectOld.(*appsv1.StatefulSet)
                newSS := e.ObjectNew.(*appsv1.StatefulSet)
                // Reconcile on spec changes (Generation) OR status changes (ReadyReplicas)
                // Without checking ReadyReplicas, Database status would never update to Ready!
                return oldSS.Generation != newSS.Generation ||
                    oldSS.Status.ReadyReplicas != newSS.Status.ReadyReplicas
            },
            CreateFunc: func(e event.CreateEvent) bool {
                return true
            },
            DeleteFunc: func(e event.DeleteEvent) bool {
                return true
            },
        })).
        Owns(&corev1.Service{}).
        Watches(
            &corev1.Secret{},
            handler.EnqueueRequestsFromMapFunc(r.findDatabasesForSecret),
        ).
        Complete(r)
}
```

## Упражнение 5: тестирование производительности отслеживания

### Задача 5.1: создайте несколько ресурсов

```bash
# Install and run operator
make install
make run

# Create multiple Databases
for i in {1..10}; do
  kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: db-$i
spec:
  image: postgres:14
  replicas: 1
  databaseName: db$i
  username: admin
  storage:
    size: 10Gi
EOF
done
```

### Задача 5.2: понаблюдайте за поведением отслеживания

```bash
# Watch operator logs
# Should see efficient reconciliation

# Update one Database
kubectl patch database db-5 --type merge -p '{"spec":{"replicas":2}}'

# Only db-5 should be reconciled
```

## Очистка

```bash
# Delete all test resources
kubectl delete databases --all
```

## Итоги лабораторной

В этой лабораторной вы:
- Настроили отслеживание подчинённых ресурсов
- Отследили неподчинённые ресурсы
- Создали индексы для эффективного поиска
- Добавили предикаты событий
- Протестировали производительность отслеживания

## Ключевые уроки

1. Отслеживайте подчинённые ресурсы с помощью `Owns()`
2. Отслеживайте неподчинённые ресурсы с помощью `Watches()`
3. Индексы обеспечивают быстрый поиск
4. Предикаты событий фильтруют события
5. Отслеживание делает контроллеры реактивными
6. Правильное отслеживание повышает производительность

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Watch Setup](../solutions/watch-setup.go) — примеры настройки отслеживания подчинённых и неподчинённых ресурсов

## Дальнейшие шаги

Теперь давайте реализуем продвинутые паттерны, такие как многофазное согласование!

**Навигация:** [← Предыдущая лабораторная: Финализаторы](lab-02-finalizers-cleanup.md) | [Связанный урок](../lessons/03-watching-indexing.md) | [Следующая лабораторная: Продвинутые паттерны →](lab-04-advanced-patterns.md)
