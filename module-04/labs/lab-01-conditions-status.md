---
layout: default
title: "Lab 04.1: Conditions Status"
nav_order: 11
parent: "Модуль 4: Продвинутое согласование"
grand_parent: Модули
mermaid: true
---

# Лабораторная 4.1: Реализация условий статуса

**Связанный урок:** [Урок 4.1: Условия и управление статусом](../lessons/01-conditions-status.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: Финализаторы →](lab-02-finalizers-cleanup.md)

## Цели

- Добавить условия (conditions) в ваш оператор Database
- Реализовать вспомогательные функции для условий
- Обновлять условия на основе состояния ресурса
- Наблюдать за переходами условий

## Предварительные требования

- Завершение [Модуля 3](../../module-03/README.md)
- Оператор PostgreSQL из Модуля 3
- Понимание управления статусом

## Упражнение 1: добавление условий в status

### Задача 1.1: обновите тип Status

Отредактируйте `api/v1/database_types.go`:

```go
// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
	// Phase is the current phase
	// +kubebuilder:validation:Enum=Pending;Creating;Ready;Failed
	Phase string `json:"phase,omitempty"`

	// Ready indicates if the database is ready
	Ready bool `json:"ready,omitempty"`

	// Endpoint is the database endpoint
	Endpoint string `json:"endpoint,omitempty"`

	// SecretName is the name of the Secret containing database credentials
	SecretName string `json:"secretName,omitempty"`

	// Conditions represent the latest observations
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// ObservedGeneration tracks the generation this status applies to
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}
```

### Задача 1.2: перегенерируйте код

```bash
# Regenerate code
make generate
make manifests
```

## Упражнение 2: реализация вспомогательных функций для условий

### Задача 2.1: добавьте вспомогательные функции

Добавьте в `internal/controller/database_controller.go`:

```go
import (
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// setCondition sets a condition on the Database
func (r *DatabaseReconciler) setCondition(db *databasev1.Database, conditionType string, status metav1.ConditionStatus, reason, message string) {
    condition := metav1.Condition{
        Type:               conditionType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        LastTransitionTime: metav1.Now(),
        ObservedGeneration: db.Generation,
    }
    
    meta.SetStatusCondition(&db.Status.Conditions, condition)
}

// getCondition gets a condition by type
func (r *DatabaseReconciler) getCondition(db *databasev1.Database, conditionType string) *metav1.Condition {
    return meta.FindStatusCondition(db.Status.Conditions, conditionType)
}
```

## Упражнение 3: обновление логики согласования

### Задача 3.1: добавьте условия в Reconcile

Измените свои функции `reconcileStatefulSet` и `updateStatus`, как показано ниже:

```go
func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *databasev1.Database) error {
	// ... existing code

	if errors.IsNotFound(err) {
		// Set owner reference
		if err := ctrl.SetControllerReference(db, desiredStatefulSet, r.Scheme); err != nil {
			return err
		}
		logger.Info("Creating StatefulSet", "name", desiredStatefulSet.Name)
		r.setCondition(db, "Ready", metav1.ConditionFalse, "StatefulSetNotFound", "StatefulSet not found")
		r.setCondition(db, "Progressing", metav1.ConditionTrue, "Creating", "Creating StatefulSet")
		return r.Create(ctx, desiredStatefulSet)
	} else if err != nil {
		r.setCondition(db, "Ready", metav1.ConditionFalse, "Error", err.Error())
		return err
	}

	// ... existing code

	return nil
}

func (r *DatabaseReconciler) updateStatus(ctx context.Context, db *databasev1.Database) error {
	// ... existing code
    
	if err != nil {
		db.Status.Phase = "Pending"
		db.Status.Ready = false
	} else {
		if statefulSet.Status.ReadyReplicas == *statefulSet.Spec.Replicas {
			db.Status.Phase = "Ready"
			db.Status.Ready = true
			db.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local:5432", db.Name, db.Namespace)
			r.setCondition(db, "Ready", metav1.ConditionTrue, "AllReplicasReady", "All replicas are ready")
			r.setCondition(db, "Progressing", metav1.ConditionFalse, "ReconciliationComplete", "Reconciliation complete")
		} else {
			db.Status.Phase = "Creating"
			db.Status.Ready = false
			r.setCondition(db, "Ready", metav1.ConditionFalse, "ReplicasNotReady",
				fmt.Sprintf("%d/%d replicas ready", statefulSet.Status.ReadyReplicas, *statefulSet.Spec.Replicas))
			r.setCondition(db, "Progressing", metav1.ConditionTrue, "Scaling", "Waiting for replicas to be ready")
		}
	}

	return r.Status().Update(ctx, db)
}
```

## Упражнение 4: тестирование условий

### Задача 4.1: установите и запустите

```bash
# Install CRD
make install

# Run operator
make run
```

### Задача 4.2: создайте Database

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
```

### Задача 4.3: понаблюдайте за условиями

```bash
# Watch conditions
kubectl get database test-db -o jsonpath='{.status.conditions}' | jq '.'

# Watch condition transitions
watch -n 1 'kubectl get database test-db -o jsonpath="{.status.conditions[?(@.type==\"Ready\")]}"'
```

## Упражнение 5: тестирование переходов условий

### Задача 5.1: масштабируйте базу данных

```bash
# Scale up
kubectl patch database test-db --type merge -p '{"spec":{"replicas":3}}'

# Watch Progressing condition
kubectl get database test-db -o jsonpath='{.status.conditions[?(@.type=="Progressing")]}'
```

### Задача 5.2: проверьте наблюдаемое поколение

```bash
# Get generation
kubectl get database test-db -o jsonpath='{.metadata.generation}'

# Get observed generation
kubectl get database test-db -o jsonpath='{.status.conditions[0].observedGeneration}'

# They should match when reconciliation is complete
```

## Очистка

```bash
# Delete Database
kubectl delete database test-db
```

## Итоги лабораторной

В этой лабораторной вы:
- Добавили условия в статус Database
- Реализовали вспомогательные функции для условий
- Обновили условия в согласовании
- Наблюдали за переходами условий
- Протестировали обновления условий

## Ключевые уроки

1. Условия предоставляют структурированное сообщение о статусе
2. Используйте meta.SetStatusCondition для обновлений
3. Отслеживайте наблюдаемое поколение
4. Обновляйте условия на основе фактического состояния
5. Условия переходят между состояниями
6. Стандартные типы условий улучшают UX

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Condition Helpers](../solutions/conditions-helpers.go) — вспомогательные функции для управления условиями

## Дальнейшие шаги

Теперь давайте реализуем финализаторы для аккуратной очистки!

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-conditions-status.md) | [Следующая лабораторная: Финализаторы →](lab-02-finalizers-cleanup.md)
