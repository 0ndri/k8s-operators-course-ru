---
layout: default
title: "Lab 03.1: Controller Runtime"
nav_order: 11
parent: "Модуль 3: Создание кастомных контроллеров"
grand_parent: Модули
mermaid: true
---

# Лабораторная 3.1: Исследование Controller Runtime

**Связанный урок:** [Урок 3.1: Глубокое погружение в Controller Runtime](../lessons/01-controller-runtime.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: Проектирование API →](lab-02-designing-api.md)

## Цели

- Изучить архитектуру controller-runtime
- Понять настройку Manager
- Реализовать разные сценарии повтора (requeue)
- Отследить вызовы согласования

## Предварительные требования

- Завершение [Модуля 2](../module-02/README.md)
- Запущенный кластер kind
- Понимание базовой структуры оператора

## Упражнение 1: изучение настройки Manager

### Задача 1.1: просмотрите ваш оператор Hello World

```bash
# Navigate to your hello-world-operator from Module 2
cd ~/hello-world-operator

# Examine main.go
cat main.go
```

**Вопросы:**
1. Как создаётся Manager?
2. Какие опции настроены?
3. Как настраивается реконсайлер?

### Задача 1.2: разберитесь в опциях Manager

```go
// In main.go, examine the Manager options:
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    Scheme:                 scheme,
    MetricsBindAddress:     metricsAddr,
    Port:                   9443,
    HealthProbeBindAddress: probeAddr,
    LeaderElection:          enableLeaderElection,
})
```

**Вопросы:**
1. Что делает каждая опция?
2. Почему выбор лидера важен?
3. Каково назначение метрик и проб здоровья (health probes)?

## Упражнение 2: изучение функции Reconcile

### Задача 2.1: изучите текущую функцию Reconcile

```bash
# Look at your controller
cat internal/controller/helloworld_controller.go
```

**Обратите внимание:**
- Сигнатуру функции
- Как она читает ресурсы
- Как она возвращает результаты
- Обработку ошибок

### Задача 2.2: добавьте логирование

Добавьте подробное логирование, чтобы понять процесс:

```go
func (r *HelloWorldReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)
    
    log.Info("Reconcile called", "request", req)
    
    // ... existing code ...
    
    log.Info("Reconcile completed", "result", result)
    return result, err
}
```

## Упражнение 3: реализация разных сценариев повтора

### Задача 3.1: немедленный повтор

Измените ваш контроллер, чтобы повторять немедленно при определённых условиях:

```go
// If ConfigMap is being created, requeue to check status
if !configMapCreated {
    log.Info("ConfigMap not ready, requeuing")
    return ctrl.Result{Requeue: true}, nil
}
```

### Задача 3.2: отложенный повтор

Добавьте отложенный повтор для ограничения частоты:

```go
// If external dependency is not ready, check again in 10 seconds
if !dependencyReady {
    log.Info("Dependency not ready, requeuing in 10s")
    return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
}
```

### Задача 3.3: без повтора

Убедитесь, что успешные случаи не вызывают повтор:

```go
// Everything is in desired state
log.Info("Reconciliation successful")
return ctrl.Result{}, nil
```

## Упражнение 4: трассировка вызовов согласования

### Задача 4.1: запустите оператор с подробным логированием

```bash
# Run operator
make run

# In another terminal, create a resource
kubectl apply -f - <<EOF
apiVersion: hello.example.com/v1
kind: HelloWorld
metadata:
  name: trace-test
spec:
  message: "Trace me"
  count: 3
EOF
```

**Обратите внимание:**
- Когда вызывается Reconcile
- Какой запрос передаётся
- Какой результат возвращается
- Как часто он вызывается

### Задача 4.2: измените ресурс и понаблюдайте

```bash
# Update the resource
kubectl patch helloworld trace-test --type merge -p '{"spec":{"count":5}}'

# Watch logs - see reconciliation triggered
```

## Упражнение 5: понимание использования клиента

### Задача 5.1: изучите операции клиента

В вашем контроллере найдите:
- Вызовы `r.Get()`
- Вызовы `r.Create()`
- Вызовы `r.Update()`
- Вызовы `r.Status().Update()`

### Задача 5.2: добавьте обработку ошибок клиента

Улучшите обработку ошибок:

```go
// Get resource with proper error handling
if err := r.Get(ctx, req.NamespacedName, helloWorld); err != nil {
    if errors.IsNotFound(err) {
        log.Info("Resource not found, may have been deleted")
        return ctrl.Result{}, nil
    }
    log.Error(err, "Failed to get resource")
    return ctrl.Result{}, err
}
```

## Упражнение 6: тестирование разных сценариев

### Задача 6.1: протестируйте удаление ресурса

```bash
# Delete resource
kubectl delete helloworld trace-test

# Observe logs - see reconciliation on deletion
```

### Задача 6.2: протестируйте конкурентные обновления

```bash
# Create resource
kubectl apply -f resource.yaml

# Quickly update multiple times
kubectl patch helloworld test --type merge -p '{"spec":{"count":1}}'
kubectl patch helloworld test --type merge -p '{"spec":{"count":2}}'
kubectl patch helloworld test --type merge -p '{"spec":{"count":3}}'

# Observe how reconciliation handles rapid changes
```

## Очистка

```bash
# Delete test resources
kubectl delete helloworld trace-test test 2>/dev/null || true
```

## Итоги лабораторной

В этой лабораторной вы:
- Изучили настройку и конфигурацию Manager
- Разобрались в процессе работы функции Reconcile
- Реализовали разные стратегии повтора
- Отследили вызовы согласования
- Улучшили обработку ошибок

## Ключевые уроки

1. Manager координирует все компоненты контроллера
2. Функция Reconcile вызывается при каждом изменении ресурса
3. Разные стратегии повтора для разных сценариев
4. Client предоставляет типобезопасный доступ к ресурсам
5. Правильная обработка ошибок критически важна
6. Логирование помогает понять процесс согласования

## Дальнейшие шаги

Теперь, когда вы понимаете controller-runtime, давайте спроектируем корректный API для оператора базы данных!

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-controller-runtime.md) | [Следующая лабораторная: Проектирование API →](lab-02-designing-api.md)
