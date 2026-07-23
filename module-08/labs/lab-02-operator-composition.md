---
layout: default
title: "Lab 08.2: Operator Composition"
nav_order: 12
parent: "Модуль 8: Продвинутые темы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 8.2: Композиция операторов

**Связанный урок:** [Урок 8.2: Композиция операторов](../lessons/02-operator-composition.md)  
**Навигация:** [← Предыдущая лабораторная: Мультиарендность](lab-01-multi-tenancy.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Stateful-приложения →](lab-03-stateful-applications.md)

## Цели

- Создать зависимые операторы
- Реализовать координацию операторов
- Использовать ссылки на ресурсы
- Протестировать композицию операторов

## Предварительные требования

- Завершение [Лабораторной 8.1](lab-01-multi-tenancy.md)
- Готовый оператор Database
- Понимание зависимостей операторов

## Упражнение 1: создание оператора резервного копирования

### Задача 1.1: сгенерируйте каркас API Backup с помощью Kubebuilder

Используйте kubebuilder для генерации каркаса нового API Backup. Поскольку Backup связан с Database, мы используем ту же группу `database`:

```bash
# Navigate to your operator project
cd ~/postgres-operator

# Scaffold the Backup API (same group as Database)
kubebuilder create api \
  --group database \
  --version v1 \
  --kind Backup \
  --resource --controller

# When prompted:
# Create Resource [y/n]: y
# Create Controller [y/n]: y
```

> **Примечание:** мы используем `--group database` (как у Database), потому что оба ресурса — часть одного оператора. Использование другой группы потребовало бы включения многогрупповой компоновки. См. [документацию kubebuilder по нескольким группам](https://kubebuilder.io/migration/multi-group.html), если вам нужны отдельные группы.

Это создаёт:
- `api/v1/backup_types.go` — определения типов API
- `internal/controller/backup_controller.go` — каркас контроллера

### Задача 1.2: определите Spec и Status Backup

Отредактируйте сгенерированный `api/v1/backup_types.go`, добавив поля spec и status:

```go
package v1

import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// BackupSpec defines the desired state of Backup
type BackupSpec struct {
    // DatabaseRef references the Database to backup
    // +kubebuilder:validation:Required
    DatabaseRef corev1.LocalObjectReference `json:"databaseRef"`

    // Schedule is the cron schedule for automated backups (optional)
    // +optional
    Schedule string `json:"schedule,omitempty"`

    // Retention is the number of backups to retain
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:default=5
    // +optional
    Retention int `json:"retention,omitempty"`
}

// BackupStatus defines the observed state of Backup
type BackupStatus struct {
    // Phase is the current backup phase
    // +kubebuilder:validation:Enum=Pending;InProgress;Completed;Failed
    Phase string `json:"phase,omitempty"`

    // BackupTime is when the backup was created
    BackupTime *metav1.Time `json:"backupTime,omitempty"`

    // BackupLocation is where the backup is stored
    BackupLocation string `json:"backupLocation,omitempty"`

    // Conditions represent the latest observations
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Database",type="string",JSONPath=".spec.databaseRef.name"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// Backup is the Schema for the backups API
type Backup struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   BackupSpec   `json:"spec,omitempty"`
    Status BackupStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// BackupList contains a list of Backup
type BackupList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Backup `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Backup{}, &BackupList{})
}
```

### Задача 1.3: сгенерируйте и установите CRD

```bash
# Generate code and CRD manifests
make generate
make manifests

# Install CRDs
make install

# Verify the CRD was created (same group as Database)
kubectl get crd backups.database.example.com
```

### Задача 1.4: реализуйте контроллер Backup

Контроллеру Backup нужно несколько функций для корректной работы. Вместо написания с нуля скопируйте полную реализацию из файла решений:

```bash
# Copy the complete controller implementation
cp path/to/solutions/backup-operator.go internal/controller/backup_controller.go
```

Или, если предпочитаете набрать сами, скопируйте полный контроллер из:
**[solutions/backup-operator.go](../solutions/backup-operator.go)**

Полный контроллер включает:
- `Reconcile()` — основной цикл согласования (показан ниже)
- `performBackup()` — обновляет статус и запускает резервное копирование
- `createBackup()` — выполняет фактическую операцию резервного копирования
- `SetupWithManager()` — регистрирует контроллер в менеджере

**Ключевая логика согласования:**

```go
func (r *BackupReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    backup := &databasev1.Backup{}
    if err := r.Get(ctx, req.NamespacedName, backup); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // Skip if already completed
    if backup.Status.Phase == "Completed" {
        return ctrl.Result{}, nil
    }
    
    // Get Database
    db := &databasev1.Database{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      backup.Spec.DatabaseRef.Name,
        Namespace: backup.Namespace,
    }, db)
    
    if errors.IsNotFound(err) {
        // Database not found - set Pending status and wait
        backup.Status.Phase = "Pending"
        r.Status().Update(ctx, backup)
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    if err != nil {
        return ctrl.Result{}, err
    }
    
    // Check if database is ready
    if db.Status.Phase != "Ready" {
        // Database not ready - set Pending status and wait
        backup.Status.Phase = "Pending"
        r.Status().Update(ctx, backup)
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }
    
    // Perform backup
    return r.performBackup(ctx, db, backup)
}
```

### Задача 1.5: соберите и проверьте

```bash
# Generate code (deep copy methods, etc.)
make generate

# Generate manifests (CRDs, RBAC from kubebuilder markers)
make manifests

# Ensure the code compiles
make build

# If there are any compilation errors, verify you copied the complete
# controller from the solutions file
```

Команда `make manifests` генерирует правила RBAC из маркеров `+kubebuilder:rbac` в контроллере, создавая необходимые разрешения ClusterRole.

## Упражнение 2: координация операторов

### Задача 2.1: добавьте ссылку на Backup в Database

Обновите существующий `api/v1/database_types.go`, добавив поле BackupRef в DatabaseSpec:

```go
type DatabaseSpec struct {
    // ... existing fields ...

    // BackupRef references a Backup resource that manages backups for this database.
    // When set, the Database controller will coordinate with the Backup controller.
    // +optional
    BackupRef *corev1.LocalObjectReference `json:"backupRef,omitempty"`
}
```

После добавления поля перегенерируйте манифесты:

```bash
make generate manifests
```

### Задача 2.2: проверка статуса Backup

Контроллер Database использует паттерн конечного автомата. Добавьте вспомогательную функцию для проверки статуса резервной копии, затем интегрируйте её в поток согласования.

Сначала добавьте вспомогательную функцию в `internal/controller/database_controller.go`:

```go
// checkBackupStatus checks if the referenced Backup is ready
func (r *DatabaseReconciler) checkBackupStatus(ctx context.Context, db *databasev1.Database) (bool, error) {
    if db.Spec.BackupRef == nil {
        // No backup reference, proceed
        return true, nil
    }

    logger := log.FromContext(ctx)
    backup := &databasev1.Backup{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Spec.BackupRef.Name,
        Namespace: db.Namespace,
    }, backup)

    if errors.IsNotFound(err) {
        logger.Info("Backup not found, waiting", "backup", db.Spec.BackupRef.Name)
        return false, nil
    }
    if err != nil {
        return false, err
    }

    // Check if backup is completed
    if backup.Status.Phase != "Completed" {
        logger.Info("Waiting for backup to complete", 
            "backup", db.Spec.BackupRef.Name, 
            "phase", backup.Status.Phase)
        return false, nil
    }

    return true, nil
}
```

Затем интегрируйте её в функцию `reconcileWithStateMachine` (перед switch по состояниям):

```go
func (r *DatabaseReconciler) reconcileWithStateMachine(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    currentState := DatabaseState(db.Status.Phase)
    if currentState == "" {
        currentState = StatePending
    }

    logger := log.FromContext(ctx)
    logger.Info("Reconciling", "state", currentState)

    // Check backup status before proceeding (if BackupRef is set)
    if currentState == StatePending || currentState == StateProvisioning {
        ready, err := r.checkBackupStatus(ctx, db)
        if err != nil {
            return ctrl.Result{}, err
        }
        if !ready {
            r.setCondition(db, "Progressing", metav1.ConditionFalse, 
                "WaitingForBackup", "Waiting for backup to be ready")
            r.Status().Update(ctx, db)
            return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
        }
    }

    switch currentState {
    // ... existing state handlers ...
    }
}
```

Не забудьте добавить маркер RBAC для чтения ресурсов Backup:

```go
// +kubebuilder:rbac:groups=database.example.com,resources=backups,verbs=get;list;watch
```

После внесения изменений перегенерируйте манифесты:

```bash
make generate manifests
```

## Упражнение 3: использование условий статуса

Условия статуса предоставляют стандартизированный способ для операторов сообщать о состоянии. Это упражнение показывает, как контроллер Backup устанавливает условия, а контроллер Database их читает.

### Задача 3.1: установите условие в контроллере Backup

Отредактируйте `internal/controller/backup_controller.go`, чтобы устанавливать условия при завершении резервного копирования:

```go
func (r *BackupReconciler) performBackup(ctx context.Context, db *databasev1.Database, backup *databasev1.Backup) (ctrl.Result, error) {
    // Perform backup...
    
    // Set condition
    meta.SetStatusCondition(&backup.Status.Conditions, metav1.Condition{
        Type:    "BackupReady",
        Status:  metav1.ConditionTrue,
        Reason:  "BackupCompleted",
        Message: "Backup completed successfully",
    })
    
    backup.Status.Phase = "Completed"
    return ctrl.Result{}, r.Status().Update(ctx, backup)
}
```

> **Примечание:** если вы скопировали полный контроллер из `solutions/backup-operator.go`, это уже реализовано.

### Задача 3.2: проверьте условие в контроллере Database

Это улучшенная версия функции `checkBackupStatus` из Задачи 2.2. Вместо проверки `Phase` она использует стандартизированный паттерн `Condition`, который предоставляет более детальную информацию о состоянии.

Обновите функцию `checkBackupStatus` в `internal/controller/database_controller.go`, чтобы использовать условия:

```go
// checkBackupStatus checks if the referenced Backup is ready using conditions
func (r *DatabaseReconciler) checkBackupStatus(ctx context.Context, db *databasev1.Database) (bool, error) {
    if db.Spec.BackupRef == nil {
        return true, nil
    }

    logger := log.FromContext(ctx)
    backup := &databasev1.Backup{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Spec.BackupRef.Name,
        Namespace: db.Namespace,
    }, backup)

    if errors.IsNotFound(err) {
        logger.Info("Backup not found, waiting", "backup", db.Spec.BackupRef.Name)
        return false, nil
    }
    if err != nil {
        return false, err
    }

    // Use condition instead of Phase for more robust checking
    condition := meta.FindStatusCondition(backup.Status.Conditions, "BackupReady")
    if condition == nil || condition.Status != metav1.ConditionTrue {
        logger.Info("Waiting for backup condition to be ready",
            "backup", db.Spec.BackupRef.Name,
            "condition", condition)
        return false, nil
    }

    return true, nil
}
```

Эта функция уже интегрирована в `reconcileWithStateMachine` из Задачи 2.2, поэтому дополнительные изменения не нужны.

## Упражнение 4: тестирование композиции операторов

### Задача 4.1: соберите и разверните оператор в кластер Kind

Соберите и разверните оператор с новым контроллером Backup:

```bash
# Build the container image
make docker-build IMG=postgres-operator:latest

# Load image into kind cluster
kind load docker-image postgres-operator:latest --name k8s-operators-course
```

Перед развёртыванием убедитесь, что в `config/manager/manager.yaml` установлено `imagePullPolicy: IfNotPresent`:

```yaml
containers:
- name: manager
  image: controller:latest
  imagePullPolicy: IfNotPresent  # Add this line if not present
```

Теперь разверните:

```bash
# Deploy operator to cluster
make deploy IMG=postgres-operator:latest

# Verify operator is running
kubectl get pods -n postgres-operator-system

# Check logs (in a separate terminal or background)
kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager -f
```

> **Используете Podman вместо Docker?**
> 
> ```bash
> # Build with podman
> make docker-build IMG=postgres-operator:latest CONTAINER_TOOL=podman
> 
> # Load image into kind (save to tarball, then load)
> podman save localhost/postgres-operator:latest -o /tmp/postgres-operator.tar
> kind load image-archive /tmp/postgres-operator.tar --name k8s-operators-course
> rm /tmp/postgres-operator.tar
> 
> # Deploy with localhost/ prefix
> make deploy IMG=localhost/postgres-operator:latest
> ```

> **Получаете `ErrImagePull` или `ImagePullBackOff`?**
> 
> Убедитесь, что установлено `imagePullPolicy: IfNotPresent` и имя образа совпадает с загруженным в kind.

Перезапустите развёртывание, если вы используете существующий кластер kind из предыдущих лабораторных, где оператор уже был развёрнут:

```
# restart the deployment to pickup newly pushed image
kubectl rollout restart deploy -n postgres-operator-system   postgres-operator-controller-manager

# check status of the deployment
kubectl rollout status deploy -n postgres-operator-system   postgres-operator-controller-manager
```
### Задача 4.2: создайте Database и Backup

Сначала создайте Database. Backup будет ждать, пока он станет готов:

```bash
# Create Database first
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
    size: "1Gi"
EOF

# Create Backup (references the Database)
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Backup
metadata:
  name: my-database-backup
spec:
  databaseRef:
    name: my-database
  schedule: "0 2 * * *"
EOF
```

> **Примечание:** Backup ссылается на Database через `databaseRef`. Контроллер Backup будет ждать, пока Database станет Ready, перед выполнением резервного копирования. Поле `backupRef` в Database (из Задачи 2.1) опционально и используется для продвинутых сценариев, таких как восстановление перед провижинингом.

### Задача 4.3: проверьте координацию

```bash
# Check Database status
kubectl get database my-database -o yaml

# Check Backup status
kubectl get backup my-database-backup -o yaml

# Verify operators coordinate (check operator logs)
kubectl logs -n postgres-operator-system -l control-plane=controller-manager | grep -i backup
```

## Очистка

```bash
# Delete test resources
kubectl delete databases --all
kubectl delete backups --all
```

## Итоги лабораторной

В этой лабораторной вы:
- Сгенерировали каркас нового API Backup с помощью kubebuilder
- Реализовали оператор резервного копирования с логикой координации
- Использовали ссылки на ресурсы между операторами
- Протестировали композицию операторов

## Ключевые уроки

1. **Используйте kubebuilder для генерации каркаса новых API** — `kubebuilder create api` берёт на себя шаблонный код
2. **Операторы могут зависеть друг от друга** — Backup зависит от Database
3. **Ссылки на ресурсы связывают операторы** — `DatabaseRef` соединяет Backup с Database
4. **Условия статуса координируют состояние** — условие `BackupReady` для проверок между операторами
5. **Управление зависимостями важно** — ждите зависимости перед продолжением
6. **Композиция позволяет создавать сложные приложения** — несколько операторов работают вместе

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Backup Types](../solutions/backup_types.go) — полные определения типов API
- [Backup Operator](../solutions/backup-operator.go) — полный контроллер резервного копирования
- [Operator Coordination](../solutions/operator-coordination.go) — примеры координации

## Дальнейшие шаги

Теперь давайте изучим управление stateful-приложениями!

**Навигация:** [← Предыдущая лабораторная: Мультиарендность](lab-01-multi-tenancy.md) | [Связанный урок](../lessons/02-operator-composition.md) | [Следующая лабораторная: Stateful-приложения →](lab-03-stateful-applications.md)
