---
layout: default
title: "Lab 04.4: Advanced Patterns"
nav_order: 14
parent: "Модуль 4: Продвинутое согласование"
grand_parent: Модули
mermaid: true
---

# Лабораторная 4.4: Многофазное согласование

**Связанный урок:** [Урок 4.4: Продвинутые паттерны](../lessons/04-advanced-patterns.md)  
**Навигация:** [← Предыдущая лабораторная: Отслеживание](lab-03-watching-indexing.md) | [Обзор модуля](../README.md)

## Цели

- Реализовать многофазное согласование
- Создать конечный автомат для оператора Database
- Обработать внешние зависимости
- Обеспечить идемпотентность

## Предварительные требования

- Завершение [Лабораторной 4.3](lab-03-watching-indexing.md)
- Оператор Database с отслеживанием
- Понимание продвинутых паттернов

## Упражнение 1: реализация конечного автомата

### Задача 1.0: обновите типы API (важно!)

Прежде чем реализовывать конечный автомат, нужно обновить валидацию поля `Phase` в типах API, чтобы разрешить новые состояния.

Отредактируйте `api/v1/database_types.go` и обновите enum поля Phase:

```go
// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
    // Phase is the current phase
    // +kubebuilder:validation:Enum=Pending;Provisioning;Configuring;Deploying;Verifying;Ready;Failed
    Phase string `json:"phase,omitempty"`
    
    // ... rest of status fields
}
```

Затем перегенерируйте и переустановите CRD:

```bash
make manifests
make install
```

> **Важно:** если вы пропустите этот шаг, вы увидите ошибки валидации вроде:
> `Database.database.example.com "test-db" is invalid: phase: Unsupported value: "Provisioning": supported values: "Pending", "Creating", "Ready", "Failed"`

### Задача 1.1: определите состояния

Добавьте константы состояний в `internal/controller/database_controller.go`:

```go
type DatabaseState string

const (
    StatePending     DatabaseState = "Pending"
    StateProvisioning DatabaseState = "Provisioning"
    StateConfiguring  DatabaseState = "Configuring"
    StateDeploying    DatabaseState = "Deploying"
    StateVerifying    DatabaseState = "Verifying"
    StateReady        DatabaseState = "Ready"
    StateFailed       DatabaseState = "Failed"
)
```

### Задача 1.2: реализуйте конечный автомат

```go
func (r *DatabaseReconciler) reconcileWithStateMachine(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    currentState := DatabaseState(db.Status.Phase)
    if currentState == "" {
        currentState = StatePending
    }
    
    logger := log.FromContext(ctx)
    logger.Info("Reconciling", "state", currentState)
    
    switch currentState {
    case StatePending:
        return r.transitionToProvisioning(ctx, db)
    case StateProvisioning:
        return r.handleProvisioning(ctx, db)
    case StateConfiguring:
        return r.handleConfiguring(ctx, db)
    case StateDeploying:
        return r.handleDeploying(ctx, db)
    case StateVerifying:
        return r.handleVerifying(ctx, db)
    case StateReady:
        return r.handleReady(ctx, db)
    case StateFailed:
        return r.handleFailed(ctx, db)
    default:
        return ctrl.Result{}, nil
    }
}
```

### Задача 1.3: обновите основную функцию Reconcile

**Важно:** вы должны обновить свою основную функцию `Reconcile`, чтобы она вызывала конечный автомат вместо прямого потока согласования:

```go
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Read Database resource
    db := &databasev1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
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
        return r.handleDeletion(ctx, db)
    }

    logger.Info("Reconciling Database", "name", db.Name)

    // Use state machine for multi-phase reconciliation
    return r.reconcileWithStateMachine(ctx, db)
}
```

> **Примечание:** если вы пропустите этот шаг и оставите старую функцию Reconcile, которая напрямую вызывает `reconcileStatefulSet`, `reconcileService` и `updateStatus`, функции конечного автомата никогда не будут вызваны, и вы увидите только переходы `Pending → Creating → Ready`.

## Упражнение 2: реализация обработчиков состояний

### Задача 2.1: функции переходов состояний

```go
func (r *DatabaseReconciler) transitionToProvisioning(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("STATE TRANSITION: Pending -> Provisioning", "database", db.Name)

    db.Status.Phase = string(StateProvisioning)
    db.Status.Ready = false
    r.setCondition(db, "Progressing", metav1.ConditionTrue, "Provisioning", "Starting provisioning")
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    logger.Info("Waiting 15 seconds before next reconciliation (for visualization)", "currentPhase", db.Status.Phase)
    // Delay to visualize state transition (remove in production)
    return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
}

func (r *DatabaseReconciler) handleProvisioning(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Handling Provisioning phase", "database", db.Name)

    // Ensure Secret exists first (StatefulSet needs it for credentials)
    if err := r.reconcileSecret(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    // Check if StatefulSet exists
    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet)
    
    if errors.IsNotFound(err) {
        // Create StatefulSet
        logger.Info("Creating StatefulSet", "database", db.Name)
        if err := r.reconcileStatefulSet(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }
    
    // StatefulSet exists, move to next phase
    logger.Info("STATE TRANSITION: Provisioning -> Configuring", "database", db.Name)
    db.Status.Phase = string(StateConfiguring)
    r.setCondition(db, "Progressing", metav1.ConditionTrue, "Configuring", "StatefulSet created, configuring")
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    logger.Info("Waiting 15 seconds before next reconciliation (for visualization)", "currentPhase", db.Status.Phase)
    // Delay to visualize state transition (remove in production)
    return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
}

func (r *DatabaseReconciler) handleConfiguring(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Handling Configuring phase", "database", db.Name)

    // Ensure Service exists
    logger.Info("Creating Service", "database", db.Name)
    if err := r.reconcileService(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    // Configure database (create users, databases, etc.)
    // For now, just move to next phase
    logger.Info("STATE TRANSITION: Configuring -> Deploying", "database", db.Name)
    db.Status.Phase = string(StateDeploying)
    r.setCondition(db, "Progressing", metav1.ConditionTrue, "Deploying", "Configuration complete, deploying")
    if err := r.Status().Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    logger.Info("Waiting 15 seconds before next reconciliation (for visualization)", "currentPhase", db.Status.Phase)
    // Delay to visualize state transition (remove in production)
    return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
}

func (r *DatabaseReconciler) handleDeploying(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Handling Deploying phase", "database", db.Name)

    // Check if StatefulSet is ready
    statefulSet := &appsv1.StatefulSet{}
    if err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet); err != nil {
        return ctrl.Result{}, err
    }
    
    if statefulSet.Status.ReadyReplicas == *statefulSet.Spec.Replicas {
        logger.Info("STATE TRANSITION: Deploying -> Verifying", "database", db.Name)
        db.Status.Phase = string(StateVerifying)
        r.setCondition(db, "Progressing", metav1.ConditionTrue, "Verifying", "Deployment complete, verifying")
        if err := r.Status().Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }

        logger.Info("Waiting 15 seconds before next reconciliation (for visualization)", "currentPhase", db.Status.Phase)
        // Delay to visualize state transition (remove in production)
        return ctrl.Result{RequeueAfter: 15 * time.Second}, nil
    }
    
    // Not ready yet
    logger.Info("Waiting for StatefulSet replicas to be ready",
        "database", db.Name,
        "readyReplicas", statefulSet.Status.ReadyReplicas,
        "desiredReplicas", *statefulSet.Spec.Replicas)
    return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
}

func (r *DatabaseReconciler) handleVerifying(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Handling Verifying phase", "database", db.Name)

    // Verify database is working (connect, run test query, etc.)
    // For now, assume it's ready
    logger.Info("STATE TRANSITION: Verifying -> Ready", "database", db.Name)
    db.Status.Phase = string(StateReady)
    db.Status.Ready = true
    db.Status.SecretName = r.secretName(db)
    db.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local:5432", db.Name, db.Namespace)
    r.setCondition(db, "Ready", metav1.ConditionTrue, "AllChecksPassed", "Database is ready")
    r.setCondition(db, "Progressing", metav1.ConditionFalse, "ReconciliationComplete", "Reconciliation complete")

    logger.Info("Database is now READY!", "database", db.Name, "endpoint", db.Status.Endpoint)
    return ctrl.Result{}, r.Status().Update(ctx, db)
}

func (r *DatabaseReconciler) handleReady(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // Monitor and maintain ready state
    // Check if updates are needed
    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) handleFailed(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // Handle failed state
    // Could retry or wait for manual intervention
    return ctrl.Result{RequeueAfter: 1 * time.Minute}, nil
}
```

> **Примечание:** задержки `RequeueAfter: 15 * time.Second` и вызовы `logger.Info()` добавлены, чтобы помочь визуализировать переходы состояний во время разработки. Смотрите одновременно логи оператора и статус Database, чтобы увидеть каждый переход. В продакшене эти задержки следует убрать и снизить подробность логирования.

## Упражнение 3: тестирование конечного автомата

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

# Watch phase transitions
watch -n 1 'kubectl get database test-db -o jsonpath="{.status.phase}"'
```

### Задача 3.3: понаблюдайте за переходами состояний

Откройте **два терминала**, чтобы наблюдать за конечным автоматом в действии:

**Терминал 1 — смотрите логи оператора:**
```bash
# The operator logs will show STATE TRANSITION messages like:
# STATE TRANSITION: Pending -> Provisioning
# STATE TRANSITION: Provisioning -> Configuring
# etc.

# If running with `make run`, logs appear in that terminal
# Look for lines containing "STATE TRANSITION" and "Waiting 15 seconds"
```

**Терминал 2 — смотрите статус Database:**
```bash
# Watch phase transitions (updates every second)
watch -n 1 'kubectl get database test-db -o jsonpath="{.status.phase}"'

# Or watch the full status including conditions
kubectl get database test-db -o jsonpath='{.status.conditions}' | jq '.'
```

**Ожидаемая последовательность состояний (каждая фаза видна ~15 секунд):**
```
Pending -> Provisioning -> Configuring -> Deploying -> Verifying -> Ready
```

**Ожидаемый вывод логов:**
```
INFO    STATE TRANSITION: Pending -> Provisioning    {"database": "test-db"}
INFO    Waiting 15 seconds before next reconciliation (for visualization)    {"currentPhase": "Provisioning"}
INFO    Handling Provisioning phase    {"database": "test-db"}
INFO    STATE TRANSITION: Provisioning -> Configuring    {"database": "test-db"}
...
INFO    Database is now READY!    {"database": "test-db", "endpoint": "test-db.default.svc.cluster.local:5432"}
```

## Упражнение 4: обработка внешних зависимостей

### Задача 4.1: добавьте проверку внешней зависимости

```go
func (r *DatabaseReconciler) checkExternalDependency(ctx context.Context, db *databasev1.Database) error {
    // Simulate external dependency check
    // In real operator, this would check external API, service, etc.
    
    // For demo, always return success
    return nil
}

func (r *DatabaseReconciler) handleProvisioning(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // Check external dependency before proceeding
    if err := r.checkExternalDependency(ctx, db); err != nil {
        r.setCondition(db, "Ready", metav1.ConditionFalse, "ExternalDependencyUnavailable", err.Error())
        r.Status().Update(ctx, db)
        // Retry after delay
        return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
    }
    
    // Proceed with provisioning
    // ...
}
```

## Упражнение 5: обеспечение идемпотентности

### Задача 5.1: разберите идемпотентные операции

Ваша существующая функция `reconcileStatefulSet` уже следует идемпотентному паттерну. Разберём, как она работает:

```go
func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *databasev1.Database) error {
    logger := log.FromContext(ctx)

    // Step 1: Get current state
    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet)

    // Step 2: Build desired state
    desiredStatefulSet := r.buildStatefulSet(db)

    // Step 3: Create if not exists (idempotent - won't fail if already exists)
    if errors.IsNotFound(err) {
        if err := ctrl.SetControllerReference(db, desiredStatefulSet, r.Scheme); err != nil {
            return err
        }
        logger.Info("Creating StatefulSet", "name", desiredStatefulSet.Name)
        return r.Create(ctx, desiredStatefulSet)
    } else if err != nil {
        return err
    }

    // Step 4: Update only if different (idempotent - won't update if already correct)
    if statefulSet.Spec.Replicas != desiredStatefulSet.Spec.Replicas {
        return r.patchStatefulSetReplicas(ctx, statefulSet, *desiredStatefulSet.Spec.Replicas)
    }

    if statefulSet.Spec.Template.Spec.Containers[0].Image != desiredStatefulSet.Spec.Template.Spec.Containers[0].Image {
        statefulSet.Spec = desiredStatefulSet.Spec
        logger.Info("Updating StatefulSet", "name", statefulSet.Name)
        return r.updateWithRetry(ctx, statefulSet, 3)
    }

    // Step 5: Already in desired state - do nothing (idempotent)
    return nil
}
```

### Ключевые принципы идемпотентности

1. **Проверка перед созданием**: всегда проверяйте существование ресурса перед созданием
2. **Сравнение перед обновлением**: обновляйте только если фактическое состояние отличается от желаемого
3. **Используйте патчи, когда возможно**: `patchStatefulSetReplicas` точнее, чем полные обновления
4. **Обрабатывайте конфликты**: `updateWithRetry` обрабатывает конфликты одновременного изменения
5. **Без побочных эффектов при no-op**: если состояние уже корректно, функция сразу возвращается

## Очистка

```bash
# Delete test resources
kubectl delete databases --all
```

## Итоги лабораторной

В этой лабораторной вы:
- Реализовали многофазное согласование
- Создали конечный автомат
- Обработали внешние зависимости
- Обеспечили идемпотентность
- Протестировали переходы состояний

## Ключевые уроки

1. Многофазное согласование обрабатывает сложные развёртывания
2. Конечные автоматы обеспечивают структурированные переходы
3. Внешние зависимости требуют проверок доступности
4. Все операции должны быть идемпотентными
5. Переходы состояний должны быть чёткими и наблюдаемыми
6. Обработка ошибок критически важна в конечных автоматах

## Решения

Эта лабораторная объединяет концепции из предыдущих лабораторных. См.:
- [State Machine Controller](../solutions/state-machine-controller.go) — **полная реализация конечного автомата**
- [Condition Helpers](../solutions/conditions-helpers.go) — для управления статусом
- [Finalizer Handler](../solutions/finalizer-handler.go) — для паттернов очистки
- [Watch Setup](../solutions/watch-setup.go) — для паттернов отслеживания

> **Примечание:** файл `state-machine-controller.go` содержит полную реализацию, включая обновлённую функцию `Reconcile`, которая вызывает конечный автомат. Убедитесь, что ваша основная функция `Reconcile` вызывает `reconcileWithStateMachine`, а не согласовывает ресурсы напрямую.

## Поздравляем!

Вы завершили Модуль 4! Теперь вы понимаете:
- Управление статусом с помощью условий
- Финализаторы для очистки
- Отслеживание и индексирование
- Продвинутые паттерны согласования

В Модуле 5 вы изучите вебхуки для валидации и мутации!

**Навигация:** [← Предыдущая лабораторная: Отслеживание](lab-03-watching-indexing.md) | [Связанный урок](../lessons/04-advanced-patterns.md) | [Обзор модуля](../README.md)
