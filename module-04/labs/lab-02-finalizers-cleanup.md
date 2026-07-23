---
layout: default
title: "Lab 04.2: Finalizers Cleanup"
nav_order: 12
parent: "Модуль 4: Продвинутое согласование"
grand_parent: Модули
mermaid: true
---

# Лабораторная 4.2: Реализация финализаторов

**Связанный урок:** [Урок 4.2: Финализаторы и очистка](../lessons/02-finalizers-cleanup.md)  
**Навигация:** [← Предыдущая лабораторная: Условия](lab-01-conditions-status.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Отслеживание →](lab-03-watching-indexing.md)

## Цели

- Добавить финализаторы в оператор Database
- Реализовать логику очистки
- Обработать аккуратное удаление
- Протестировать сценарии очистки

## Предварительные требования

- Завершение [Лабораторной 4.1](lab-01-conditions-status.md)
- Оператор Database с условиями
- Понимание финализаторов

## Упражнение 1: добавление финализатора при создании

### Задача 1.1: добавьте логику финализатора

Измените функцию `Reconcile`, чтобы добавить финализатор:

```go
import (
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

const (
    finalizerName := "database.example.com/finalizer"
)

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    db := &databasev1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, err
    }
    
    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(db, finalizerName) {
        controllerutil.AddFinalizer(db, finalizerName)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        logger.Info("Added finalizer", "name", db.Name)
    }
    
    // Check if resource is being deleted
    if !db.DeletionTimestamp.IsZero() {
        // Resource is being deleted
        return r.handleDeletion(ctx, db)
    }
    
    // Normal reconciliation
    // ... existing reconciliation logic ...
}
```

## Упражнение 2: реализация логики очистки

### Задача 2.1: создайте функцию очистки

Добавьте функцию очистки:

```go
func (r *DatabaseReconciler) handleDeletion(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    // Check if finalizer exists
    if !controllerutil.ContainsFinalizer(db, finalizerName) {
        return ctrl.Result{}, nil
    }
    
    logger.Info("Handling deletion", "name", db.Name)
    
    // Perform cleanup operations
    if err := r.cleanupExternalResources(ctx, db); err != nil {
        logger.Error(err, "Failed to cleanup external resources")
        r.setCondition(db, "Ready", metav1.ConditionFalse, "CleanupFailed", err.Error())
        r.Status().Update(ctx, db)
        // Retry after delay
        return ctrl.Result{RequeueAfter: 10 * time.Second}, err
    }
    
    // Cleanup successful, remove finalizer
    controllerutil.RemoveFinalizer(db, finalizerName)
    if err := r.Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    logger.Info("Finalizer removed, resource will be deleted")
    return ctrl.Result{}, nil
}
```

### Задача 2.2: реализуйте очистку

```go
func (r *DatabaseReconciler) cleanupExternalResources(ctx context.Context, db *databasev1.Database) error {
    logger := log.FromContext(ctx)
    
    // Delete StatefulSet if it exists
    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet)
    
    if err == nil {
        // StatefulSet exists, delete it
        logger.Info("Deleting StatefulSet", "name", statefulSet.Name)
        if err := r.Delete(ctx, statefulSet); err != nil && !errors.IsNotFound(err) {
            return fmt.Errorf("failed to delete StatefulSet: %w", err)
        }
        // Requeue to wait for deletion to complete
        return fmt.Errorf("waiting for StatefulSet to be deleted")
    } else if !errors.IsNotFound(err) {
        // Some other error occurred
        return fmt.Errorf("failed to get StatefulSet: %w", err)
    }
    
    // StatefulSet is gone, now cleanup Service
    service := &corev1.Service{}
    err = r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, service)
    
    if err == nil {
        logger.Info("Deleting Service", "name", service.Name)
        if err := r.Delete(ctx, service); err != nil && !errors.IsNotFound(err) {
            return fmt.Errorf("failed to delete Service: %w", err)
        }
        return fmt.Errorf("waiting for Service to be deleted")
    } else if !errors.IsNotFound(err) {
        return fmt.Errorf("failed to get Service: %w", err)
    }
    
    // Cleanup Secret
    secret := &corev1.Secret{}
    err = r.Get(ctx, client.ObjectKey{
        Name:      r.secretName(db),
        Namespace: db.Namespace,
    }, secret)
    
    if err == nil {
        logger.Info("Deleting Secret", "name", secret.Name)
        if err := r.Delete(ctx, secret); err != nil && !errors.IsNotFound(err) {
            return fmt.Errorf("failed to delete Secret: %w", err)
        }
        return fmt.Errorf("waiting for Secret to be deleted")
    } else if !errors.IsNotFound(err) {
        return fmt.Errorf("failed to get Secret: %w", err)
    }
    
    // Example: Delete backup in external system
    // if err := r.deleteBackup(ctx, db); err != nil {
    //     return err
    // }
    
    logger.Info("Cleanup completed")
    return nil
}
```

> **Важно:** функция очистки должна **явно удалять** дочерние ресурсы. Хотя ссылки-владельцы включают автоматическую сборку мусора при удалении родителя, финализаторы не дают удалить родителя, пока не завершится очистка. Это создаёт взаимоблокировку, если вы просто ждёте исчезновения ресурсов, — их нужно активно удалять.

## Упражнение 3: тестирование финализаторов

### Задача 3.1: установите и запустите

```bash
# Install CRD
make install

# Run operator
make run
```

### Задача 3.2: создайте Database

```bash
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

# Verify finalizer was added
kubectl get database test-db -o jsonpath='{.metadata.finalizers}'
```

### Задача 3.3: удалите Database

```bash
# Delete Database
kubectl delete database test-db

# Check deletion timestamp, note you may or may not see it, as deletion is pretty quick but you can track the log messages in the terminal you have run the operator
kubectl get database test-db -o jsonpath='{.metadata.deletionTimestamp}'

# Resource should still exist (has finalizer), it may or may not exist, as deletion is pretty quick
kubectl get database test-db

# Watch operator logs - should see cleanup
```

### Задача 3.4: проверьте очистку

```bash
# Watch finalizer removal
watch -n 1 'kubectl get database test-db -o jsonpath="{.metadata.finalizers}"'

# After cleanup, resource should be deleted
kubectl get database test-db
```

## Упражнение 4: тестирование сбоя очистки

### Задача 4.1: имитируйте сбой очистки

Временно измените очистку, чтобы она всегда завершалась ошибкой:

```go
func (r *DatabaseReconciler) cleanupExternalResources(ctx context.Context, db *databasev1.Database) error {
    return fmt.Errorf("simulated cleanup failure")
}
```

Перезапустите следующие команды, чтобы использовать код с имитацией сбоя:

```bash
# Install CRD
make install

# Run operator
make run
```

### Задача 4.2: протестируйте поведение

```bash
# Create and delete Database
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

# the delete command should hang as finalizer cannot be removed
kubectl delete database test-db

# Resource should remain (cleanup failing)
kubectl get database test-db

# Check conditions, run 
kubectl get database test-db -o jsonpath='{.status.conditions}' | jq .
```

**Верните исходный код очистки и перезапустите оператор — база данных должна корректно очиститься.**

## Упражнение 5: понимание идемпотентной очистки

**Идемпотентная** означает, что очистку можно вызывать несколько раз с одним и тем же результатом — она не завершится ошибкой и не вызовет проблем, если ресурсы уже удалены.

### Задача 5.1: разберите паттерны идемпотентности

Наша функция `cleanupExternalResources` из Задачи 2.2 уже идемпотентна! Вот почему:

```go
func (r *DatabaseReconciler) cleanupExternalResources(ctx context.Context, db *databasev1.Database) error {
    logger := log.FromContext(ctx)
    
    // Pattern 1: Check existence before deleting
    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet)
    
    if err == nil {
        // Only delete if it exists
        logger.Info("Deleting StatefulSet", "name", statefulSet.Name)
        // Pattern 2: Ignore "not found" errors on delete
        if err := r.Delete(ctx, statefulSet); err != nil && !errors.IsNotFound(err) {
            return fmt.Errorf("failed to delete StatefulSet: %w", err)
        }
        return fmt.Errorf("waiting for StatefulSet to be deleted")
    } else if !errors.IsNotFound(err) {
        // Only fail on unexpected errors, not "not found"
        return fmt.Errorf("failed to get StatefulSet: %w", err)
    }
    
    // Pattern 3: If we reach here, resource is already gone - that's OK!
    // ... continue with next resource ...
    
    logger.Info("Cleanup completed")
    return nil
}
```

**Используемые ключевые паттерны идемпотентности:**

1. **Проверка перед удалением**: используйте `Get()`, чтобы проверить существование ресурса перед попыткой удаления
2. **Игнорирование NotFound при удалении**: `!errors.IsNotFound(err)` — если уже удалён, это нормально
3. **NotFound как успех**: если ресурс не существует, очистка для этого ресурса завершена

### Задача 5.2: протестируйте идемпотентность

Запустите очистку несколько раз, чтобы проверить идемпотентность:

```bash
# Create a Database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: idempotent-test
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Wait for it to be ready
kubectl wait --for=condition=Ready database/idempotent-test --timeout=60s

# Delete the Database
kubectl delete database idempotent-test

# Watch operator logs - cleanup should succeed even if called multiple times
# You'll see logs like "Cleanup completed" without errors
```

### Задача 5.3: неидемпотентный антипаттерн (не делайте так!)

Вот как выглядит **неидемпотентная** очистка — избегайте этого:

```go
// BAD: Non-idempotent cleanup - will fail on second call
func (r *DatabaseReconciler) badCleanup(ctx context.Context, db *databasev1.Database) error {
    statefulSet := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
    }
    
    // BAD: This will return an error if StatefulSet doesn't exist
    if err := r.Delete(ctx, statefulSet); err != nil {
        return err  // Fails on "not found" - not idempotent!
    }
    
    return nil
}
```

Проблема: если контроллер перезапустится в середине очистки или цикл reconcile выполнится снова, это завершится ошибкой, потому что StatefulSet уже удалён.

## Очистка

```bash
# Delete any remaining resources
kubectl delete databases --all
```

## Итоги лабораторной

В этой лабораторной вы:
- Добавили финализаторы в оператор Database
- Реализовали логику очистки
- Обработали аккуратное удаление
- Протестировали сценарии очистки
- Сделали очистку идемпотентной

## Ключевые уроки

1. Финализаторы предотвращают удаление, пока не завершится очистка
2. Добавляйте финализатор в начале согласования
3. Проверяйте DeletionTimestamp для обнаружения удаления
4. **Явно удаляйте дочерние ресурсы** — не полагайтесь на каскад ссылок-владельцев во время очистки финализатором (это вызывает взаимоблокировку)
5. Выполняйте очистку до удаления финализатора
6. Делайте очистку идемпотентной
7. Аккуратно обрабатывайте сбои очистки

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Finalizer Handler](../solutions/finalizer-handler.go) — полная реализация финализатора с логикой очистки

## Дальнейшие шаги

Теперь давайте настроим отслеживание и индексы для эффективных контроллеров!

**Навигация:** [← Предыдущая лабораторная: Условия](lab-01-conditions-status.md) | [Связанный урок](../lessons/02-finalizers-cleanup.md) | [Следующая лабораторная: Отслеживание →](lab-03-watching-indexing.md)
