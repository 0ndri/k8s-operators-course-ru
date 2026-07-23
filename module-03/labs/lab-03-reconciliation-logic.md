---
layout: default
title: "Lab 03.3: Reconciliation Logic"
nav_order: 13
parent: "Модуль 3: Создание кастомных контроллеров"
grand_parent: Модули
mermaid: true
---

# Лабораторная 3.3: Создание оператора PostgreSQL

**Связанный урок:** [Урок 3.3: Реализация логики согласования](../lessons/03-reconciliation-logic.md)  
**Навигация:** [← Предыдущая лабораторная: Проектирование API](lab-02-designing-api.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Client-Go →](lab-04-client-go.md)

## Цели

- Реализовать логику согласования для оператора PostgreSQL
- Обрабатывать создание и обновление ресурсов
- Использовать ссылки-владельцы
- Управлять Secret для учётных данных базы данных
- Протестировать идемпотентность

## Предварительные требования

- Завершение [Лабораторной 3.2](lab-02-designing-api.md)
- Определённый API Database
- Понимание паттернов согласования

## Упражнение 1: реализация базового согласования

### Задача 1.1: настройте структуру контроллера

Отредактируйте `internal/controller/database_controller.go`:

```go
package controller

import (
	"context"
	"crypto/rand"
	"encoding/base64"
	"fmt"

	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	databasev1 "github.com/example/postgres-operator/api/v1"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// DatabaseReconciler reconciles a Database object
type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=database.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.example.com,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create;update;patch;delete

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
    
    logger.Info("Reconciling Database", "name", db.Name)
    
    // Reconcile Secret (must be done before StatefulSet)
    if err := r.reconcileSecret(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    // Reconcile StatefulSet
    if err := r.reconcileStatefulSet(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    // Reconcile Service
    if err := r.reconcileService(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    // Update status
    if err := r.updateStatus(ctx, db); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

## Упражнение 2: реализация управления Secret

Контроллер автоматически генерирует случайный пароль и хранит его в Secret Kubernetes.
Это безопаснее, чем требовать от пользователей указывать пароли в открытом виде.

### Задача 2.1: вспомогательные функции

Добавьте вспомогательные функции для управления Secret:

```go
// secretName returns the name of the Secret for this Database
func (r *DatabaseReconciler) secretName(db *databasev1.Database) string {
	return fmt.Sprintf("%s-credentials", db.Name)
}

// generatePassword generates a random password
func generatePassword(length int) (string, error) {
	bytes := make([]byte, length)
	if _, err := rand.Read(bytes); err != nil {
		return "", err
	}
	return base64.URLEncoding.EncodeToString(bytes)[:length], nil
}
```

### Задача 2.2: согласование Secret

```go
// reconcileSecret ensures the credentials Secret exists
func (r *DatabaseReconciler) reconcileSecret(ctx context.Context, db *databasev1.Database) error {
	logger := log.FromContext(ctx)
	secretName := r.secretName(db)

	secret := &corev1.Secret{}
	err := r.Get(ctx, client.ObjectKey{
		Name:      secretName,
		Namespace: db.Namespace,
	}, secret)

	if errors.IsNotFound(err) {
		// Generate random password
		password, err := generatePassword(16)
		if err != nil {
			return fmt.Errorf("failed to generate password: %w", err)
		}

		// Create new secret
		secret = &corev1.Secret{
			ObjectMeta: metav1.ObjectMeta{
				Name:      secretName,
				Namespace: db.Namespace,
			},
			Type: corev1.SecretTypeOpaque,
			StringData: map[string]string{
				"username": db.Spec.Username,
				"password": password,
				"database": db.Spec.DatabaseName,
			},
		}

		// Set owner reference
		if err := ctrl.SetControllerReference(db, secret, r.Scheme); err != nil {
			return err
		}

		logger.Info("Creating Secret", "name", secretName)
		return r.Create(ctx, secret)
	} else if err != nil {
		return err
	}

	// Secret already exists, don't update password
	return nil
}
```

## Упражнение 3: реализация согласования StatefulSet

### Задача 3.1: постройте StatefulSet

Добавьте вспомогательную функцию для построения StatefulSet. Обратите внимание, как мы ссылаемся на пароль из Secret:

```go
func (r *DatabaseReconciler) buildStatefulSet(db *databasev1.Database) *appsv1.StatefulSet {
	replicas := int32(1)
	if db.Spec.Replicas != nil {
		replicas = *db.Spec.Replicas
	}

	image := db.Spec.Image
	if image == "" {
		image = "postgres:14"
	}

	secretName := r.secretName(db)

	return &appsv1.StatefulSet{
		ObjectMeta: metav1.ObjectMeta{
			Name:      db.Name,
			Namespace: db.Namespace,
		},
		Spec: appsv1.StatefulSetSpec{
			Replicas: &replicas,
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app":      "database",
					"database": db.Name,
				},
			},
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app":      "database",
						"database": db.Name,
					},
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Name:  "postgres",
							Image: image,
							Env: []corev1.EnvVar{
								{
									Name:  "POSTGRES_DB",
									Value: db.Spec.DatabaseName,
								},
								{
									Name: "POSTGRES_USER",
									ValueFrom: &corev1.EnvVarSource{
										SecretKeyRef: &corev1.SecretKeySelector{
											LocalObjectReference: corev1.LocalObjectReference{
												Name: secretName,
											},
											Key: "username",
										},
									},
								},
								{
									Name: "POSTGRES_PASSWORD",
									ValueFrom: &corev1.EnvVarSource{
										SecretKeyRef: &corev1.SecretKeySelector{
											LocalObjectReference: corev1.LocalObjectReference{
												Name: secretName,
											},
											Key: "password",
										},
									},
								},
								{
									Name:  "PGDATA",
									Value: "/var/lib/postgresql/data/pgdata",
								},
							},
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "data",
									MountPath: "/var/lib/postgresql/data",
								},
							},
						},
					},
				},
			},
			VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
				{
					ObjectMeta: metav1.ObjectMeta{
						Name: "data",
					},
					Spec: corev1.PersistentVolumeClaimSpec{
						AccessModes: []corev1.PersistentVolumeAccessMode{
							corev1.ReadWriteOnce,
						},
						Resources: corev1.VolumeResourceRequirements{
							Requests: corev1.ResourceList{
								corev1.ResourceStorage: resource.MustParse(db.Spec.Storage.Size),
							},
						},
					},
				},
			},
		},
	}
}
```

### Задача 3.2: согласование StatefulSet

```go
func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *databasev1.Database) error {
	logger := log.FromContext(ctx)

	statefulSet := &appsv1.StatefulSet{}
	err := r.Get(ctx, client.ObjectKey{
		Name:      db.Name,
		Namespace: db.Namespace,
	}, statefulSet)

	desiredStatefulSet := r.buildStatefulSet(db)

	if errors.IsNotFound(err) {
		// Set owner reference
		if err := ctrl.SetControllerReference(db, desiredStatefulSet, r.Scheme); err != nil {
			return err
		}
		logger.Info("Creating StatefulSet", "name", desiredStatefulSet.Name)
		return r.Create(ctx, desiredStatefulSet)
	} else if err != nil {
		return err
	}

	// Update if needed
	if statefulSet.Spec.Replicas != desiredStatefulSet.Spec.Replicas ||
		statefulSet.Spec.Template.Spec.Containers[0].Image != desiredStatefulSet.Spec.Template.Spec.Containers[0].Image {
		statefulSet.Spec = desiredStatefulSet.Spec
		logger.Info("Updating StatefulSet", "name", statefulSet.Name)
		return r.Update(ctx, statefulSet)
	}

	return nil
}
```

## Упражнение 4: реализация согласования Service

### Задача 4.1: постройте Service

```go
func (r *DatabaseReconciler) buildService(db *databasev1.Database) *corev1.Service {
    return &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
        Spec: corev1.ServiceSpec{
            Selector: map[string]string{
                "app":      "database",
                "database": db.Name,
            },
            Ports: []corev1.ServicePort{
                {
                    Port: 5432,
                    Name: "postgres",
                },
            },
        },
    }
}
```

### Задача 4.2: согласование Service

```go
func (r *DatabaseReconciler) reconcileService(ctx context.Context, db *databasev1.Database) error {
    service := &corev1.Service{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, service)
    
    desiredService := r.buildService(db)
    
    if errors.IsNotFound(err) {
        if err := ctrl.SetControllerReference(db, desiredService, r.Scheme); err != nil {
            return err
        }
        return r.Create(ctx, desiredService)
    } else if err != nil {
        return err
    }
    
    // Service updates are less common, but handle if needed
    return nil
}
```

## Упражнение 5: обновление статуса

### Задача 5.1: реализуйте обновление статуса

Статус включает имя Secret, чтобы пользователи знали, где найти учётные данные:

```go
func (r *DatabaseReconciler) updateStatus(ctx context.Context, db *databasev1.Database) error {
    // Set the secret name in status
    db.Status.SecretName = r.secretName(db)

    // Check StatefulSet status
    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet)
    
    if err != nil {
        db.Status.Phase = "Pending"
        db.Status.Ready = false
    } else {
        if statefulSet.Status.ReadyReplicas == *statefulSet.Spec.Replicas {
            db.Status.Phase = "Ready"
            db.Status.Ready = true
            db.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local:5432", db.Name, db.Namespace)
        } else {
            db.Status.Phase = "Creating"
            db.Status.Ready = false
        }
    }
    
    return r.Status().Update(ctx, db)
}
```

## Упражнение 6: настройка Controller Manager

Чтобы контроллер получал события при изменении подчинённых (owned) ресурсов (например, когда StatefulSet становится готовым), нужно указать менеджеру отслеживать эти ресурсы.

### Задача 6.1: настройте отслеживание

Добавьте функцию `SetupWithManager` в конце вашего контроллера:

```go
// SetupWithManager sets up the controller with the Manager.
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&databasev1.Database{}).
		Owns(&appsv1.StatefulSet{}).
		Owns(&corev1.Service{}).
		Owns(&corev1.Secret{}).
		Complete(r)
}
```

**Ключевые моменты:**
- `For(&databasev1.Database{})` — отслеживает ресурсы Database (основной ресурс)
- `Owns(&appsv1.StatefulSet{})` — отслеживает StatefulSet, принадлежащие Database (через ссылку-владельца)
- `Owns(&corev1.Service{})` — отслеживает Service, принадлежащие Database
- `Owns(&corev1.Secret{})` — отслеживает Secret, принадлежащие Database

Это гарантирует, что при изменении статуса StatefulSet (поды становятся готовы) контроллер получает уведомление и согласовывает родительский Database, чтобы обновить его статус.

## Упражнение 7: тестирование оператора

### Задача 7.1: установите и запустите

```bash
# Install CRD
make install

# Run operator
make run
```

### Задача 7.2: создайте Database

```bash
# Create Database resource (no password needed - it's auto-generated!)
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
```

### Задача 7.3: наблюдайте за согласованием

```bash
# Watch Database status
kubectl get database my-database -w

# Check StatefulSet
kubectl get statefulset my-database

# Check Service
kubectl get service my-database

# Check the auto-generated Secret
kubectl get secret my-database-credentials

# View the generated password (base64 decoded)
kubectl get secret my-database-credentials -o jsonpath='{.data.password}' | base64 -d

# Check operator logs
```

## Упражнение 8: тестирование идемпотентности

### Задача 8.1: примените несколько раз

```bash
# Apply the same resource multiple times
for i in {1..3}; do
  kubectl apply -f database.yaml
  sleep 2
done

# Verify only one StatefulSet exists
kubectl get statefulsets | grep my-database
```

### Задача 8.2: протестируйте обновления

```bash
# Update replicas
kubectl patch database my-database --type merge -p '{"spec":{"replicas":2}}'

# Verify StatefulSet was updated
kubectl get statefulset my-database -o jsonpath='{.spec.replicas}'
```

## Очистка

```bash
# Delete Database (should cascade delete StatefulSet, Service, and Secret)
kubectl delete database my-database

# Verify resources were deleted
kubectl get statefulset my-database
kubectl get service my-database
kubectl get secret my-database-credentials
```

## Итоги лабораторной

В этой лабораторной вы:
- Реализовали полную логику согласования
- Создали Secret с автоматически сгенерированным паролем
- Создали StatefulSet и Service
- Использовали ссылки-владельцы для всех ресурсов
- Настроили отслеживание с помощью `Owns()`, чтобы реагировать на изменения подчинённых ресурсов
- Обновили статус именем Secret
- Протестировали идемпотентность
- Проверили каскадное удаление

## Ключевые уроки

1. Согласование следует схеме: чтение, сравнение, создание/обновление, статус
2. Ссылки-владельцы обеспечивают каскадное удаление
3. **Используйте `Owns()` для отслеживания подчинённых ресурсов** — без этого контроллер не получит уведомление при изменении статуса StatefulSet/Service/Secret
4. Идемпотентность критически важна
5. Secret должны генерироваться автоматически, а не предоставляться пользователем в открытом виде
6. Обновления статуса отражают фактическое состояние и предоставляют полезную информацию (например, имя Secret)
7. Обработка ошибок важна
8. Логирование помогает при отладке

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Database Types](../solutions/database-types.go) — полные определения типов API Database
- [Database Controller](../solutions/database-controller.go) — полный контроллер с согласованием Secret/StatefulSet/Service

## Дальнейшие шаги

Теперь давайте изучим продвинутые операции клиента для более совершенных контроллеров!

**Навигация:** [← Предыдущая лабораторная: Проектирование API](lab-02-designing-api.md) | [Связанный урок](../lessons/03-reconciliation-logic.md) | [Следующая лабораторная: Client-Go →](lab-04-client-go.md)
