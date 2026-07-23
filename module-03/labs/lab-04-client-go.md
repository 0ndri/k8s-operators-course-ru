---
layout: default
title: "Lab 03.4: Client Go"
nav_order: 14
parent: "Модуль 3: Создание кастомных контроллеров"
grand_parent: Модули
mermaid: true
---

# Лабораторная 3.4: Продвинутые операции клиента

**Связанный урок:** [Урок 3.4: Работа с Client-Go](../lessons/04-client-go.md)  
**Навигация:** [← Предыдущая лабораторная: Логика согласования](lab-03-reconciliation-logic.md) | [Обзор модуля](../README.md)

## Цели

- Использовать продвинутые операции клиента
- Реализовать отслеживание зависимых ресурсов
- Использовать strategic merge patch
- Обрабатывать конфликты с повторами

## Предварительные требования

- Завершение [Лабораторной 3.3](lab-03-reconciliation-logic.md)
- Оператор PostgreSQL из предыдущей лабораторной
- Понимание операций клиента

## Упражнение 1: получение списка ресурсов с фильтрами

### Задача 1.1: список по пространству имён

Добавьте функцию для получения списка всех баз данных в пространстве имён:

```go
func (r *DatabaseReconciler) listDatabasesInNamespace(ctx context.Context, namespace string) (*databasev1.DatabaseList, error) {
    list := &databasev1.DatabaseList{}
    err := r.List(ctx, list, client.InNamespace(namespace))
    return list, err
}
```

### Задача 1.2: список по меткам

```go
func (r *DatabaseReconciler) listDatabasesByLabel(ctx context.Context, labels map[string]string) (*databasev1.DatabaseList, error) {
    list := &databasev1.DatabaseList{}
    err := r.List(ctx, list, client.MatchingLabels(labels))
    return list, err
}
```

## Упражнение 2: реализация strategic merge patch

### Задача 2.1: патч реплик StatefulSet

Вместо полного обновления используйте патч:

```go
func (r *DatabaseReconciler) patchStatefulSetReplicas(ctx context.Context, statefulSet *appsv1.StatefulSet, replicas int32) error {
    patch := client.MergeFrom(statefulSet.DeepCopy())
    statefulSet.Spec.Replicas = &replicas
    return r.Patch(ctx, statefulSet, patch)
}
```

### Задача 2.2: используйте в согласовании

Обновите свою функцию reconcileStatefulSet, чтобы использовать патч, когда меняются только реплики:

```go
// If only replicas changed, use patch
if statefulSet.Spec.Replicas != desiredStatefulSet.Spec.Replicas {
    return r.patchStatefulSetReplicas(ctx, statefulSet, *desiredStatefulSet.Spec.Replicas)
}
```

## Упражнение 3: обработка конфликтов

### Задача 3.1: реализуйте логику повторов

Добавьте вспомогательную функцию для повторов при конфликтах:

```go
func (r *DatabaseReconciler) updateWithRetry(ctx context.Context, obj client.Object, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := r.Update(ctx, obj)
        if err == nil {
            return nil
        }
        
        if !errors.IsConflict(err) {
            return err
        }
        
        // Conflict - get fresh version and retry
        key := client.ObjectKeyFromObject(obj)
        if err := r.Get(ctx, key, obj); err != nil {
            return err
        }
        
        time.Sleep(100 * time.Millisecond)
    }
    return fmt.Errorf("max retries exceeded")
}
```

### Задача 3.2: используйте в согласовании

```go
// Use retry logic for updates
if err := r.updateWithRetry(ctx, statefulSet, 3); err != nil {
    return err
}
```

## Упражнение 4: отслеживание зависимых ресурсов

### Задача 4.1: настройте отслеживание

Измените SetupWithManager, чтобы отслеживать StatefulSet:

```go
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).  // Watch owned StatefulSets
        Owns(&corev1.Service{}).      // Watch owned Services
        Owns(&corev1.Secret{}).       // Watch owner Secrets
        Complete(r)
}
```

### Задача 4.2: обработка событий отслеживания

Когда StatefulSet меняется, Database будет согласован автоматически!

## Упражнение 5: селекторы полей

### Задача 5.1: найдите базы данных по владельцу

```go
func (r *DatabaseReconciler) findDatabasesByOwner(ctx context.Context, ownerName string) (*databasev1.DatabaseList, error) {
    list := &databasev1.DatabaseList{}
    err := r.List(ctx, list, client.MatchingFields{
        ".metadata.ownerReferences[0].name": ownerName,
    })
    return list, err
}
```

## Упражнение 6: тестирование продвинутых операций

### Задача 6.1: протестируйте патч

```bash
# Create database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: my-database
spec:
  image: postgres:14
  replicas: 1
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Update replicas using patch (simulate)
kubectl patch database my-database --type merge -p '{"spec":{"replicas":2}}'

# Watch operator logs to see patch in action

# Validate 2 replicas are available
kubectl get database my-database -o jsonpath='{.spec.replicas}'
kubectl get statefulset my-database
```

### Задача 6.2: протестируйте обработку конфликтов

```bash
# Quickly update multiple times to trigger conflicts
kubectl patch database my-database --type merge -p '{"spec":{"replicas":3}}'
kubectl patch database my-database --type merge -p '{"spec":{"replicas":4}}'
kubectl patch database my-database --type merge -p '{"spec":{"replicas":5}}'

# Observe how operator handles conflicts

# Validate 5 replicas are eventually available
kubectl get database my-database -o jsonpath='{.spec.replicas}'
kubectl get statefulset my-database
```

### Задача 6.3: протестируйте отслеживание

```bash
# Manually delete StatefulSet
kubectl delete statefulset my-database

# Watch operator logs - should detect and recreate
kubectl get statefulset my-database
```

## Очистка

```bash
# Delete test resources
kubectl delete databases --all

# Validate that all the resources are gone
kubectl get databases my-database
kubectl get statefulset my-database
kubectl get service my-database
kubectl get secret my-database-credentials
```

## Итоги лабораторной

В этой лабораторной вы:
- Использовали продвинутые операции получения списка с фильтрами
- Реализовали strategic merge patch
- Добавили логику повторов при конфликтах
- Настроили отслеживание зависимых ресурсов
- Использовали селекторы полей
- Протестировали все операции

## Ключевые уроки

1. Операции получения списка можно эффективно фильтровать
2. Патчи лучше подходят для частичных обновлений
3. Конфликты требуют логики повторов
4. Отслеживание обеспечивает реактивное согласование
5. Селекторы полей предоставляют мощные запросы
6. Продвинутые операции повышают эффективность оператора

## Поздравляем!

Вы завершили Модуль 3! Теперь вы понимаете:
- Архитектуру controller-runtime
- Принципы проектирования API
- Логику согласования
- Продвинутые операции клиента

В Модуле 4 вы изучите продвинутые паттерны, такие как условия (conditions), финализаторы и многофазное согласование.

**Навигация:** [← Предыдущая лабораторная: Логика согласования](lab-03-reconciliation-logic.md) | [Связанный урок](../lessons/04-client-go.md) | [Обзор модуля](../README.md)
