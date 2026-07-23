---
layout: default
title: "Lab 08.3: Stateful Applications"
nav_order: 13
parent: "Модуль 8: Продвинутые темы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 8.3: Управление stateful-приложениями

**Связанный урок:** [Урок 8.3: Управление stateful-приложениями](../lessons/03-stateful-applications.md)  
**Навигация:** [← Предыдущая лабораторная: Композиция операторов](lab-02-operator-composition.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Финальный проект →](lab-04-final-project.md)

## Цели

- Реализовать функциональность резервного копирования
- Добавить возможность восстановления
- Обрабатывать скользящие обновления
- Обеспечить согласованность данных

## Предварительные требования

- Завершение [Лабораторной 8.2](lab-02-operator-composition.md)
- Оператор Database с развёрнутым контроллером Backup
- Понимание StatefulSet

## Упражнение 1: реализация функциональности резервного копирования

В Лабораторной 8.2 мы создали базовый контроллер Backup. Теперь добавим фактическую логику резервного копирования.

### Задача 1.1: создайте пакет резервного копирования

Создайте новый пакет для операций резервного копирования. Скопируйте полную реализацию из файла решений:

```bash
# Create the backup package directory
mkdir -p internal/backup

# Copy the complete backup implementation
cp path/to/solutions/backup.go internal/backup/backup.go
```

Или, если предпочитаете набрать сами, скопируйте из:
**[solutions/backup.go](../solutions/backup.go)**

Пакет резервного копирования включает:
- `PerformBackup()` — выполняет pg_dump и сохраняет в хранилище
- `saveToStorage()` — загружает резервную копию в S3/PVC
- `PerformScheduledBackup()` — обрабатывает запланированные резервные копии

> **Важно:** реализация резервного копирования использует `pg_dump`, который требует установки клиентских инструментов PostgreSQL в контейнере вашего оператора. Вам нужно обновить Dockerfile, чтобы включить пакет `postgresql-client`. См. Задачу 1.2 ниже.

### Задача 1.2: обновите Dockerfile для клиентских инструментов PostgreSQL

> **Важное замечание о безопасности:** в [Модуле 7](../../module-07/labs/lab-01-packaging-distribution.md) мы рекомендовали использовать distroless-образы для максимальной безопасности. Однако distroless-образы не включают менеджеры пакетов, что делает невозможной прямую установку клиентских инструментов PostgreSQL (`pg_dump`, `psql`).

**Для этого курса (в учебных целях):** мы пойдём на прагматичное упрощение и используем минимальный базовый образ Debian, чтобы включить клиентские инструменты PostgreSQL. Это делает лабораторную проще и позволяет сосредоточиться на изучении функциональности резервного копирования/восстановления.

**Для продакшена:** см. готовые к продакшену альтернативы ниже, которые сохраняют безопасность и при этом предоставляют необходимые инструменты.

#### Вариант A: минимальный базовый образ Debian (упрощение для курса)

Для этого курса обновите ваш `Dockerfile`, чтобы использовать минимальный базовый образ Debian. Пример см. в [solutions/Dockerfile](../solutions/Dockerfile):

```dockerfile
# Runtime stage - use minimal Debian base instead of distroless
# NOTE: This is a shortcut for learning purposes
# For production, see Option B below
FROM debian:bookworm-slim

# Install PostgreSQL client tools
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    postgresql-client \
    ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user and group
# Using UID 65532 to match distroless images (suppress warning with --no-log-init)
RUN groupadd -r -g 65532 nonroot && \
    useradd -r -u 65532 -g nonroot --no-log-init -m -s /bin/bash nonroot

WORKDIR /
COPY --from=builder /workspace/manager .

# Switch to non-root user
USER 65532:65532

ENTRYPOINT ["/manager"]
```

#### Вариант B: готовые к продакшену подходы (рекомендуется)

Для продакшен-сред сохраняйте безопасность, используя distroless-образы и один из этих паттернов:

**1. Паттерн sidecar-контейнера**

Оставьте ваш оператор на distroless и используйте sidecar-контейнер с инструментами PostgreSQL:

```yaml
# In your operator Deployment
spec:
  template:
    spec:
      containers:
      - name: manager
        image: your-operator:distroless
        # ... operator config ...
      - name: postgres-client
        image: postgres:14-alpine
        command: ["sleep", "infinity"]
        # Mount shared volume for backup files
        volumeMounts:
        - name: backup-storage
          mountPath: /backups
```

Затем измените код резервного копирования, чтобы выполнять команды внутри sidecar-контейнера:

```go
// Execute pg_dump in sidecar container
cmd := exec.CommandContext(ctx, "kubectl", "exec", "-i", podName, 
    "-c", "postgres-client", "--", "pg_dump", ...)
```

**2. Паттерн Kubernetes Jobs**

Создавайте Kubernetes Jobs с клиентскими инструментами PostgreSQL для каждого резервного копирования:

```go
job := &batchv1.Job{
    Spec: batchv1.JobSpec{
        Template: corev1.PodTemplateSpec{
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{{
                    Name:  "backup",
                    Image: "postgres:14-alpine",
                    Command: []string{"pg_dump", ...},
                }},
            },
        },
    },
}
```

**3. Отдельный оператор резервного копирования**

Создайте выделенный оператор резервного копирования, включающий инструменты PostgreSQL, оставив основной оператор на distroless.

**4. Паттерн init-контейнера**

Используйте init-контейнер для подготовки инструментов резервного копирования, затем используйте общий том.

> **Для этого курса:** мы используем Вариант A (база Debian), чтобы упростить дело. В продакшене выберите один из подходов Варианта B в зависимости от ваших требований к безопасности и операционных предпочтений.

**Ключевая логика резервного копирования:**

```go
func PerformBackup(ctx context.Context, k8sClient client.Client, db *databasev1.Database) (string, error) {
    endpoint := db.Status.Endpoint
    if endpoint == "" {
        return "", fmt.Errorf("database endpoint not available")
    }

    // Get password from Secret
    secretName := db.Status.SecretName
    if secretName == "" {
        secretName = fmt.Sprintf("%s-credentials", db.Name)
    }

    secret := &corev1.Secret{}
    err := k8sClient.Get(ctx, client.ObjectKey{
        Name:      secretName,
        Namespace: db.Namespace,
    }, secret)
    if err != nil {
        return "", fmt.Errorf("failed to get secret: %w", err)
    }

    password := string(secret.Data["password"])

    // Create backup filename
    backupFile := fmt.Sprintf("/backups/%s-%s.sql",
        db.Name,
        time.Now().Format("20060102-150405"))

    // Perform pg_dump with password from Secret
    cmd := exec.CommandContext(ctx, "pg_dump",
        "-h", endpoint,
        "-U", db.Spec.Username,
        "-d", db.Spec.DatabaseName,
        "-f", backupFile)

    // Set password as environment variable
    cmd.Env = append(cmd.Env, fmt.Sprintf("PGPASSWORD=%s", password))

    output, err := cmd.CombinedOutput()
    if err != nil {
        return "", fmt.Errorf("backup failed: %v, output: %s", err, string(output))
    }

    // Save to storage (S3, PVC, etc.)
    return saveToStorage(backupFile)
}
```

### Задача 1.3: интегрируйте с контроллером Backup

Обновите `internal/controller/backup_controller.go`, чтобы использовать пакет резервного копирования. Контроллер Backup из Лабораторной 8.2 имеет метод `createBackup`, который сейчас имитирует резервное копирование. Замените его вызовом фактического пакета резервного копирования.

**Текущее состояние (из Лабораторной 8.2):**

Метод `createBackup` в вашем контроллере Backup сейчас выглядит так:

```go
func (r *BackupReconciler) createBackup(ctx context.Context, db *databasev1.Database, backup *databasev1.Backup) (string, error) {
    // Actual backup implementation would:
    // 1. Connect to database
    // 2. Create backup (pg_dump, mysqldump, etc.)
    // 3. Store backup in storage (S3, PVC, etc.)
    // 4. Return backup location

    backupLocation := fmt.Sprintf("s3://backups/%s/%s-%s.sql",
        db.Namespace,
        db.Name,
        time.Now().Format("20060102-150405"))

    // Simulate backup creation
    // In real implementation, this would actually perform the backup

    return backupLocation, nil
}
```

**Обновлённое состояние:**

Замените метод `createBackup`, чтобы использовать пакет резервного копирования:

```go
import (
    // ... existing imports ...
    backupPkg "github.com/example/postgres-operator/internal/backup"
)

// Replace the createBackup method to use the backup package
func (r *BackupReconciler) createBackup(ctx context.Context, db *databasev1.Database, backup *databasev1.Backup) (string, error) {
    // Use the backup package to perform actual backup
    // Note: PerformBackup requires k8sClient to retrieve password from Secret
    // Note: We use 'backupPkg' alias to avoid conflict with 'backup' variable name
    backupLocation, err := backupPkg.PerformBackup(ctx, r.Client, db)
    if err != nil {
        return "", fmt.Errorf("failed to perform backup: %w", err)
    }

    return backupLocation, nil
}
```

> **Важно:** мы используем псевдоним пакета `backupPkg`, потому что параметр функции `backup *databasev1.Backup` затенил бы имя пакета `backup`. Без псевдонима Go попытался бы вызвать `backup.PerformBackup()` на переменной ресурса Backup вместо пакета резервного копирования, вызвав ошибку компиляции: `backup.PerformBackup undefined`.

**Как это работает:**

Метод `performBackup` в вашем контроллере уже корректно обрабатывает обновления статуса. Он вызывает `createBackup` и обновляет статус Backup на основе результата:

```go
func (r *BackupReconciler) performBackup(ctx context.Context, db *databasev1.Database, backup *databasev1.Backup) (ctrl.Result, error) {
    // Update status to in progress
    backup.Status.Phase = "InProgress"
    // ... status updates ...

    // Perform actual backup (calls createBackup)
    backupLocation, err := r.createBackup(ctx, db, backup)
    if err != nil {
        // Handle error and update status
        return ctrl.Result{}, err
    }

    // Update status to completed
    backup.Status.BackupLocation = backupLocation
    // ... more status updates ...
}
```

С этим изменением `createBackup` теперь будет выполнять фактическое резервное копирование с помощью `pg_dump` вместо его имитации.

**Структура контроллера:**
- `performBackup()` — обрабатывает обновления статуса, обработку ошибок и вызывает `createBackup()`
- `createBackup()` — выполняет фактическую работу по резервному копированию (теперь вызывает `backupPkg.PerformBackup()`)

### Задача 1.4: соберите, разверните и протестируйте функциональность резервного копирования

Теперь давайте соберём и протестируем функциональность резервного копирования:

```bash
# Generate code and manifests
make generate
make manifests

# Ensure code compiles
make build

# Build the container image (with PostgreSQL client tools)
make docker-build IMG=postgres-operator:latest

# Load image into kind cluster
kind load docker-image postgres-operator:latest --name k8s-operators-course

# Deploy operator to cluster
make deploy IMG=postgres-operator:latest

# rollout restart the deployment just in case you are using existing kind cluster with operator deployed
kubectl rollout restart deploy -n postgres-operator-system postgres-operator-controller-manager
kubectl rollout status deploy -n postgres-operator-system   postgres-operator-controller-manager

# Verify operator is running
kubectl get pods -n postgres-operator-system

# Check logs
kubectl logs -n postgres-operator-system -l control-plane=controller-manager -f
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
> Убедитесь, что в `config/manager/manager.yaml` установлено `imagePullPolicy: IfNotPresent` и имя образа совпадает с загруженным в kind.

**Протестируйте функциональность резервного копирования:**

```bash
# Create a Database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: test-db
spec:
  image: postgres:14
  replicas: 1
  databaseName: testdb
  username: admin
  storage:
    size: "1Gi"
EOF

# Wait for Database to be ready
kubectl wait --for=jsonpath='{.status.phase}'=Ready database/test-db --timeout=120s

# Create a Backup
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Backup
metadata:
  name: test-backup
spec:
  databaseRef:
    name: test-db
EOF

# Watch Backup status
kubectl get backup test-backup -w

# Check Backup status details
kubectl get backup test-backup -o yaml

# Verify backup location is set
kubectl get backup test-backup -o jsonpath='{.status.backupLocation}'
```

**Убедитесь, что резервное копирование выполнено:**

```bash
# Check operator logs for backup activity
kubectl logs -n postgres-operator-system -l control-plane=controller-manager | grep -i backup

# Check Backup conditions
kubectl get backup test-backup -o jsonpath='{.status.conditions}'
```

## Упражнение 2: реализация восстановления

### Задача 2.1: сгенерируйте каркас API Restore с помощью Kubebuilder

Используйте kubebuilder для генерации каркаса нового API Restore (та же группа, что у Database и Backup):

```bash
# Navigate to your operator project
cd ~/postgres-operator

# Scaffold the Restore API
kubebuilder create api \
  --group database \
  --version v1 \
  --kind Restore \
  --resource --controller

# When prompted:
# Create Resource [y/n]: y
# Create Controller [y/n]: y
```

Это создаёт:
- `api/v1/restore_types.go` — определения типов API
- `internal/controller/restore_controller.go` — каркас контроллера

### Задача 2.2: определите Spec и Status Restore

Отредактируйте `api/v1/restore_types.go`, чтобы определить ресурс Restore:

```go
package v1

import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// RestoreSpec defines the desired state of Restore
type RestoreSpec struct {
    // DatabaseRef references the Database to restore to
    // +kubebuilder:validation:Required
    DatabaseRef corev1.LocalObjectReference `json:"databaseRef"`

    // BackupRef references the Backup to restore from
    // +kubebuilder:validation:Required
    BackupRef corev1.LocalObjectReference `json:"backupRef"`
}

// RestoreStatus defines the observed state of Restore
type RestoreStatus struct {
    // Phase is the current restore phase
    // +kubebuilder:validation:Enum=Pending;InProgress;Completed;Failed
    Phase string `json:"phase,omitempty"`

    // RestoreTime is when the restore completed
    RestoreTime *metav1.Time `json:"restoreTime,omitempty"`

    // Conditions represent the latest observations
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Database",type="string",JSONPath=".spec.databaseRef.name"
// +kubebuilder:printcolumn:name="Backup",type="string",JSONPath=".spec.backupRef.name"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// Restore is the Schema for the restores API
type Restore struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   RestoreSpec   `json:"spec,omitempty"`
    Status RestoreStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// RestoreList contains a list of Restore
type RestoreList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Restore `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Restore{}, &RestoreList{})
}
```

### Задача 2.3: создайте пакет восстановления

Создайте пакет восстановления с фактической логикой восстановления:

```bash
# Create the restore package directory
mkdir -p internal/restore

# Copy the complete restore implementation
cp path/to/solutions/restore.go internal/restore/restore.go
```

Или скопируйте из: **[solutions/restore.go](../solutions/restore.go)**

Пакет восстановления включает:
- `PerformRestore()` — загружает резервную копию и восстанавливает в базу данных
- `loadFromStorage()` — скачивает резервную копию из S3/PVC
- `stopDatabase()` / `startDatabase()` — аккуратные операции с базой данных

> **Примечание:** реализация восстановления использует `psql`, который также требует клиентских инструментов PostgreSQL. Если вы ещё не обновили Dockerfile в Задаче 1.2, сделайте это сейчас.

**Ключевая логика восстановления:**

```go
func PerformRestore(ctx context.Context, k8sClient client.Client, db *databasev1.Database, backupLocation string) error {
    // Load backup from storage
    backupData, err := loadFromStorage(backupLocation)
    if err != nil {
        return fmt.Errorf("failed to load backup: %v", err)
    }

    endpoint := db.Status.Endpoint
    if endpoint == "" {
        return fmt.Errorf("database endpoint not available")
    }

    // Get password from Secret
    secretName := db.Status.SecretName
    if secretName == "" {
        secretName = fmt.Sprintf("%s-credentials", db.Name)
    }

    secret := &corev1.Secret{}
    err = k8sClient.Get(ctx, client.ObjectKey{
        Name:      secretName,
        Namespace: db.Namespace,
    }, secret)
    if err != nil {
        return fmt.Errorf("failed to get secret: %w", err)
    }

    password := string(secret.Data["password"])

    // Perform restore using psql with password from Secret
    cmd := exec.CommandContext(ctx, "psql",
        "-h", endpoint,
        "-U", db.Spec.Username,
        "-d", db.Spec.DatabaseName)

    // Set password as environment variable
    cmd.Env = append(cmd.Env, fmt.Sprintf("PGPASSWORD=%s", password))
    cmd.Stdin = bytes.NewReader(backupData)

    output, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("restore failed: %v, output: %s", err, string(output))
    }

    return nil
}
```

### Задача 2.4: реализуйте контроллер Restore

Отредактируйте `internal/controller/restore_controller.go`, чтобы реализовать полную логику согласования.

Скопируйте полную реализацию контроллера восстановления из: **[solutions/restore-controller.go](../solutions/restore-controller.go)**

Контроллер восстановления:
- Ждёт готовности Database
- Ждёт завершения Backup
- Вызывает `restorePkg.PerformRestore()` для выполнения фактического восстановления
- Обновляет статус Restore фазами (Pending → InProgress → Completed/Failed)
- Устанавливает условия для наблюдаемости

**Ключевые детали реализации:**

```go
func (r *RestoreReconciler) performRestore(ctx context.Context, db *databasev1.Database, backup *databasev1.Backup, rst *databasev1.Restore) (ctrl.Result, error) {
    // Update status to in progress
    rst.Status.Phase = "InProgress"
    // ... status updates ...

    // Get backup location from Backup status
    if backup.Status.BackupLocation == "" {
        // Handle error
    }

    // Perform actual restore using restore package
    // Note: PerformRestore requires k8sClient to retrieve password from Secret
    err := restorePkg.PerformRestore(ctx, r.Client, db, backup.Status.BackupLocation)
    if err != nil {
        // Handle error and update status
    }

    // Update status to completed
    rst.Status.Phase = "Completed"
    rst.Status.RestoreTime = &metav1.Now()
    // ... more status updates ...
}
```

**Структура контроллера:**
- `Reconcile()` — основной цикл согласования, проверяет предварительные условия
- `performRestore()` — обрабатывает обновления статуса, обработку ошибок и вызывает `restorePkg.PerformRestore()`

### Задача 2.5: зарегистрируйте контроллер Restore

Убедитесь, что контроллер Restore зарегистрирован в `cmd/main.go`:

```go
if err = (&controller.RestoreReconciler{
    Client: mgr.GetClient(),
    Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
    setupLog.Error(err, "unable to create controller", "controller", "Restore")
    os.Exit(1)
}
```

### Задача 2.6: сгенерируйте и установите CRD

```bash
# Generate code and manifests
make generate
make manifests

# Install CRDs
make install

# Verify the CRD was created
kubectl get crd restores.database.example.com
```

### Задача 2.7: соберите, разверните и протестируйте функциональность восстановления

Теперь давайте соберём и протестируем функциональность восстановления:

```bash
# Generate code and manifests
make generate
make manifests

# Ensure code compiles
make build

# Build the container image
make docker-build IMG=postgres-operator:latest

# Load image into kind cluster
kind load docker-image postgres-operator:latest --name k8s-operators-course

# Deploy operator to cluster
make deploy IMG=postgres-operator:latest

# rollout restart the deployment just in case you are using existing kind cluster with operator deployed
kubectl rollout restart deploy -n postgres-operator-system postgres-operator-controller-manager
kubectl rollout status deploy -n postgres-operator-system   postgres-operator-controller-manager

# Verify operator is running
kubectl get pods -n postgres-operator-system

# Check logs
kubectl logs -n postgres-operator-system -l control-plane=controller-manager -f
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
> Убедитесь, что в `config/manager/manager.yaml` установлено `imagePullPolicy: IfNotPresent` и имя образа совпадает с загруженным в kind.

**Протестируйте функциональность восстановления:**

```bash
# Ensure you have a Database and completed Backup from Task 1.4
# If not, create them first:

# Create a Database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: restore-test-db
spec:
  image: postgres:14
  replicas: 1
  databaseName: restoredb
  username: admin
  storage:
    size: "1Gi"
EOF

# Wait for Database to be ready
kubectl wait --for=jsonpath='{.status.phase}'=Ready database/restore-test-db --timeout=120s

# Create a Backup
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Backup
metadata:
  name: restore-test-backup
spec:
  databaseRef:
    name: restore-test-db
EOF

# Wait for Backup to complete
kubectl wait --for=jsonpath='{.status.phase}'=Completed backup/restore-test-backup --timeout=300s

# Verify backup location exists
kubectl get backup restore-test-backup -o jsonpath='{.status.backupLocation}'
echo

# Now create a Restore
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Restore
metadata:
  name: test-restore
spec:
  databaseRef:
    name: restore-test-db
  backupRef:
    name: restore-test-backup
EOF

# Watch Restore status
kubectl get restore test-restore -w

# Check Restore status details
kubectl get restore test-restore -o yaml

# Verify restore completed successfully
kubectl get restore test-restore -o jsonpath='{.status.phase}'
echo
```

**Убедитесь, что восстановление выполнено:**

```bash
# Check operator logs for restore activity
kubectl logs -n postgres-operator-system -l control-plane=controller-manager | grep -i restore

# Check Restore conditions
kubectl get restore test-restore -o jsonpath='{.status.conditions}'
echo

# Verify restore time is set
kubectl get restore test-restore -o jsonpath='{.status.restoreTime}'
echo
```

**Протестируйте сценарии ошибок:**

```bash
# Test with non-existent database
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Restore
metadata:
  name: test-restore-fail-db
spec:
  databaseRef:
    name: non-existent-db
  backupRef:
    name: restore-test-backup
EOF

# Watch status - should stay in Pending
kubectl get restore test-restore-fail-db -w

# Test with non-existent backup
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Restore
metadata:
  name: test-restore-fail-backup
spec:
  databaseRef:
    name: restore-test-db
  backupRef:
    name: non-existent-backup
EOF

# Watch status - should stay in Pending
kubectl get restore test-restore-fail-backup -w
```

## Упражнение 3: обработка скользящих обновлений

Скользящие обновления позволяют обновлять образ базы данных без простоя. Контроллер Database уже обрабатывает базовые обновления, но это упражнение показывает продвинутые паттерны.

### Задача 3.1: изучите логику скользящего обновления

Полная реализация скользящего обновления находится в:
**[solutions/rolling-update.go](../solutions/rolling-update.go)**

Ключевые функции:
- `updateStatefulSet()` — обнаруживает изменения и обновляет StatefulSet
- `waitForRollingUpdate()` — ждёт обновления всех подов
- `createStatefulSet()` — создаёт новый StatefulSet при необходимости

**Шаг 1: добавьте вспомогательные функции**

Скопируйте полную реализацию из `solutions/rolling-update.go` в `internal/controller/database_controller.go`. Функции обрабатывают:
- Обнаружение изменений образа
- Обновление StatefulSet для запуска скользящих обновлений
- Ожидание обновления и готовности всех реплик
- Обработку изменений количества реплик

**Шаг 2: интегрируйте в логику согласования**

Глядя на текущую структуру контроллера `postgres-operator`, у вас есть паттерн конечного автомата с `handleReady()`, который уже вызывает `reconcileStatefulSet()` для обработки изменений spec. Чтобы интегрировать логику скользящего обновления с ожиданием:

**Текущее состояние:** в `handleReady()` у вас сейчас есть:

```go
func (r *DatabaseReconciler) handleReady(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Reconcile StatefulSet to handle spec changes (e.g., replica count, image)
    if err := r.reconcileStatefulSet(ctx, db); err != nil {
        logger.Error(err, "Failed to reconcile StatefulSet in Ready state")
        return ctrl.Result{}, err
    }
    // ... rest of function ...
}
```

**Интеграция:** замените вызов `reconcileStatefulSet()` на `updateStatefulSet()` в `handleReady()`:

```go
func (r *DatabaseReconciler) handleReady(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)

    // Update StatefulSet with rolling update support (waits for completion)
    // This handles image changes and replica count changes with proper waiting
    if err := r.updateStatefulSet(ctx, db); err != nil {
        logger.Error(err, "Failed to update StatefulSet in Ready state")
        return ctrl.Result{}, err
    }

    // Reconcile Service in case it was deleted
    if err := r.reconcileService(ctx, db); err != nil {
        logger.Error(err, "Failed to reconcile Service in Ready state")
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}
```

**Почему `handleReady()`?** 
- Состояние `Ready` — это место, где обрабатываются текущие изменения spec (например, обновления образа или масштабирование реплик)
- `handleProvisioning()` должен продолжать использовать `reconcileStatefulSet()` для первоначального создания
- `updateStatefulSet()` будет ждать завершения скользящих обновлений, гарантируя, что база данных полностью обновлена перед следующим согласованием

**Примечание:** функция `updateStatefulSet()`:
- Создаёт StatefulSet, если его не существует (вызывает `createStatefulSet()`)
- Обнаруживает изменения образа и запускает скользящие обновления
- Ждёт обновления и готовности всех реплик (через `waitForRollingUpdate()`)
- Обрабатывает изменения количества реплик

**Как это работает:**

1. `updateStatefulSet()` проверяет, существует ли StatefulSet, создаёт его, если нет
2. Сравнивает желаемый образ/реплики с текущим spec StatefulSet
3. Если отличается, обновляет StatefulSet (запускает скользящее обновление Kubernetes)
4. Вызывает `waitForRollingUpdate()`, чтобы дождаться обновления и готовности всех подов
5. Возвращается, когда скользящее обновление завершается или истекает таймаут

> **Примечание:** существующий контроллер Database из предыдущих модулей уже обрабатывает обновления образа. Это упражнение показывает паттерн явного ожидания для большего контроля. Функция `waitForRollingUpdate()` использует `wait.PollImmediate()` для опроса статуса StatefulSet, пока все реплики не будут обновлены и готовы, с таймаутом 5 минут.

### Задача 3.2: протестируйте скользящие обновления

Соберите и разверните обновлённый оператор:

```bash
# Ensure code compiles
make build

# Build the container image
make docker-build IMG=postgres-operator:latest

# Load image into kind cluster
kind load docker-image postgres-operator:latest --name k8s-operators-course

# Deploy operator to cluster
make deploy IMG=postgres-operator:latest

# Restart deployment if redeploying
kubectl rollout restart deploy -n postgres-operator-system postgres-operator-controller-manager
kubectl rollout status deploy -n postgres-operator-system postgres-operator-controller-manager
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
> Убедитесь, что в `config/manager/manager.yaml` установлено `imagePullPolicy: IfNotPresent` и имя образа совпадает с загруженным в kind.

**Протестируйте скользящее обновление:**

```bash
# Create a Database with initial image
# Using postgres:14 (Debian-based) as the initial image
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: rolling-update-test
spec:
  image: postgres:14
  replicas: 2
  databaseName: testdb
  username: admin
  storage:
    size: "1Gi"
EOF

# Wait for Database to be ready
kubectl wait --for=jsonpath='{.status.phase}'=Ready database/rolling-update-test --timeout=120s

# Verify initial StatefulSet pods
kubectl get pods -l app=database,database=rolling-update-test

# Check current image version
kubectl get statefulset rolling-update-test -o jsonpath='{.spec.template.spec.containers[0].image}'
echo

# Update to new image version (using compatible PostgreSQL 14 variant)
# Note: We use postgres:14-alpine instead of postgres:15 because PostgreSQL major versions
# are incompatible. Using the same major version (14) ensures data compatibility.
kubectl patch database rolling-update-test --type=merge -p '{"spec":{"image":"postgres:14-alpine"}}'

# Watch StatefulSet update
kubectl get statefulset rolling-update-test -w

# Watch pods during rolling update
kubectl get pods -l app=database,database=rolling-update-test -w

# Verify all pods are updated
kubectl get statefulset rolling-update-test -o jsonpath='{.status.updatedReplicas}/{.spec.replicas}'
echo

# Verify new image version
kubectl get statefulset rolling-update-test -o jsonpath='{.spec.template.spec.containers[0].image}'
echo

# Check operator logs for rolling update activity
kubectl logs -n postgres-operator-system -l control-plane=controller-manager | grep -i "rolling\|update"
```

**Убедитесь, что скользящее обновление завершено:**

```bash
# Check StatefulSet status
kubectl get statefulset rolling-update-test -o yaml | grep -A 10 status:

# Verify all replicas are ready
kubectl get statefulset rolling-update-test -o jsonpath='{.status.readyReplicas}/{.spec.replicas}'
echo

# Verify pods are running with new image
kubectl get pods -l app=database,database=rolling-update-test -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

## Упражнение 4: обеспечение согласованности данных

Согласованность данных критична для stateful-приложений. Это упражнение показывает паттерны проверки согласованности.

### Задача 4.1: добавьте функции проверки согласованности

Добавьте эти вспомогательные функции в `internal/controller/database_controller.go`. Вам нужно добавить `os/exec` и `strings` в импорты, если их там ещё нет:

```go
import (
    // ... existing imports ...
    "os/exec"
    "strings"
    // ... rest of imports ...
)
```

Остальные необходимые импорты (`fmt`, `log`, `client`, `appsv1`) уже должны присутствовать в файле вашего контроллера:

```go
func (r *DatabaseReconciler) ensureDataConsistency(ctx context.Context, db *databasev1.Database) error {
    logger := log.FromContext(ctx)

    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      db.Name,
        Namespace: db.Namespace,
    }, statefulSet)

    if err != nil {
        return err
    }

    // Check all replicas are ready
    if statefulSet.Spec.Replicas == nil {
        return fmt.Errorf("StatefulSet replicas not set")
    }
    
    if statefulSet.Status.ReadyReplicas != *statefulSet.Spec.Replicas {
        return fmt.Errorf("not all replicas ready: %d/%d",
            statefulSet.Status.ReadyReplicas, *statefulSet.Spec.Replicas)
    }

    logger.Info("All replicas ready, checking consistency",
        "replicas", statefulSet.Status.ReadyReplicas)

    // Perform consistency check
    return r.performConsistencyCheck(ctx, db)
}

func (r *DatabaseReconciler) performConsistencyCheck(ctx context.Context, db *databasev1.Database) error {
    logger := log.FromContext(ctx)
    
    // Verify database endpoint is set
    if db.Status.Endpoint == "" {
        return fmt.Errorf("database endpoint not available in status")
    }

    // Parse endpoint (format: hostname:port or hostname)
    host := db.Status.Endpoint
    port := "5432" // Default PostgreSQL port
    if strings.Contains(db.Status.Endpoint, ":") {
        parts := strings.Split(db.Status.Endpoint, ":")
        if len(parts) == 2 {
            host = parts[0]
            port = parts[1]
        }
    }

    // For this lab, verify database is accessible using pg_isready
    // pg_isready checks if PostgreSQL is accepting connections
    // Note: This requires PostgreSQL client tools in the operator image
    cmd := exec.CommandContext(ctx, "pg_isready",
        "-h", host,
        "-p", port)

    output, err := cmd.CombinedOutput()
    if err != nil {
        logger.Error(err, "Database accessibility check failed", 
            "endpoint", db.Status.Endpoint, 
            "output", string(output))
        return fmt.Errorf("database not accessible at %s: %v, output: %s", 
            db.Status.Endpoint, err, string(output))
    }

    logger.Info("Database accessibility check passed", 
        "endpoint", db.Status.Endpoint,
        "output", string(output))

    // In production, you would also:
    // 1. Connect to primary and verify it's writable
    // 2. Connect to replicas and verify replication lag
    // 3. Run test queries to verify data integrity
    // Example: Check replication status (PostgreSQL)
    // SELECT * FROM pg_stat_replication;

    return nil
}
```

### Задача 4.2: интегрируйте проверку согласованности

Глядя на текущий контроллер `postgres-operator`, функция `handleVerifying()` сейчас просто переходит в Ready без выполнения проверок согласованности. Обновите её, чтобы вызывать `ensureDataConsistency()`:

**Текущее состояние:** в `handleVerifying()` у вас сейчас есть:

```go
func (r *DatabaseReconciler) handleVerifying(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Handling Verifying phase", "database", db.Name)

    // Verify database is working (connect, run test query, etc.)
    // For now, assume it's ready
    logger.Info("STATE TRANSITION: Verifying -> Ready", "database", db.Name)
    db.Status.Phase = string(StateReady)
    db.Status.Ready = true
    // ... rest of status updates ...
}
```

**Интеграция:** добавьте проверку согласованности перед переходом в Ready. **Важно:** установите endpoint до вызова `ensureDataConsistency()`, поскольку проверка согласованности его использует:

```go
func (r *DatabaseReconciler) handleVerifying(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Handling Verifying phase", "database", db.Name)

    // Set endpoint and secret name before consistency check (needed for pg_isready)
    if db.Status.Endpoint == "" {
        db.Status.Endpoint = fmt.Sprintf("%s.%s.svc.cluster.local:5432", db.Name, db.Namespace)
        db.Status.SecretName = r.secretName(db)
        // Update status to persist endpoint before consistency check
        if err := r.Status().Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Verify database consistency
    if err := r.ensureDataConsistency(ctx, db); err != nil {
        logger.Info("Consistency check failed, retrying", "error", err.Error())
        return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
    }

    // All checks passed, transition to Ready
    logger.Info("STATE TRANSITION: Verifying -> Ready", "database", db.Name)
    db.Status.Phase = string(StateReady)
    db.Status.Ready = true
    r.setCondition(db, "Ready", metav1.ConditionTrue, "AllChecksPassed", "Database is ready")
    r.setCondition(db, "Progressing", metav1.ConditionFalse, "ReconciliationComplete", "Reconciliation complete")

    logger.Info("Database is now READY!", "database", db.Name, "endpoint", db.Status.Endpoint)
    r.Recorder.Event(db, "Normal", "Ready", "Database is ready at "+db.Status.Endpoint)
    return ctrl.Result{}, r.Status().Update(ctx, db)
}
```

**Как это работает:**
- `ensureDataConsistency()` проверяет, что все реплики StatefulSet готовы
- Если реплики не готовы, она возвращает ошибку, и функция повторяет попытку через 5 секунд
- Как только все реплики готовы, она вызывает `performConsistencyCheck()` для специфичных для приложения проверок
- Только когда проверки согласованности проходят, база данных переходит в состояние Ready

### Задача 4.3: протестируйте проверки согласованности данных

Соберите и разверните обновлённый оператор:

```bash
# Ensure code compiles
make build

# Build the container image
make docker-build IMG=postgres-operator:latest

# Load image into kind cluster
kind load docker-image postgres-operator:latest --name k8s-operators-course

# Deploy operator to cluster
make deploy IMG=postgres-operator:latest

# Restart deployment if redeploying
kubectl rollout restart deploy -n postgres-operator-system postgres-operator-controller-manager
kubectl rollout status deploy -n postgres-operator-system postgres-operator-controller-manager
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
> Убедитесь, что в `config/manager/manager.yaml` установлено `imagePullPolicy: IfNotPresent` и имя образа совпадает с загруженным в kind.

**Протестируйте проверки согласованности:**

```bash
# Create a Database with multiple replicas
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: consistency-test
spec:
  image: postgres:14
  replicas: 3
  databaseName: testdb
  username: admin
  storage:
    size: "1Gi"
EOF

# Watch Database status transitions
kubectl get database consistency-test -w

# Wait for Database to reach Verifying phase
kubectl wait --for=jsonpath='{.status.phase}'=Verifying database/consistency-test --timeout=120s || true

# Check Database status
kubectl get database consistency-test -o yaml | grep -A 5 status:

# Wait for Database to be Ready (consistency checks should pass)
kubectl wait --for=jsonpath='{.status.phase}'=Ready database/consistency-test --timeout=300s

# Verify all replicas are ready
kubectl get statefulset consistency-test -o jsonpath='{.status.readyReplicas}/{.spec.replicas}'
echo

# Verify pods are running
kubectl get pods -l app=database,database=consistency-test

# Check operator logs for consistency check activity
kubectl logs -n postgres-operator-system -l control-plane=controller-manager | grep -i "consistency\|replica"
```

**Протестируйте сценарий сбоя проверки согласованности:**

Чтобы проверить, как оператор обрабатывает сбои проверки согласованности, создайте базу данных с несколькими репликами и понаблюдайте за повторами проверки согласованности, пока реплики запускаются:

```bash
# Create a new database with 3 replicas
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: consistency-failure-test
spec:
  databaseName: testdb
  username: testuser
  replicas: 3
  storage:
    size: "1Gi"
EOF

# Watch Database status transitions in one terminal
kubectl get database consistency-failure-test -w

# In another terminal, watch logs for consistency check activity
kubectl logs -n postgres-operator-system -l control-plane=controller-manager -f | grep -i "consistency\|replica\|accessibility"
```

**Что наблюдать:**

1. **Во время запуска реплик**: Database перейдёт в фазу `Verifying`, как только StatefulSet начнёт развёртывать поды
2. **Повторы проверки согласованности**: вы должны увидеть логи вроде:
   - `"Consistency check failed, retrying"` с ошибкой `"not all replicas ready: 1/3"` (или похожей)
   - Контроллер будет повторять каждые 5 секунд (как настроено в `handleVerifying`)
3. **Как только все реплики готовы**: проверка `ensureDataConsistency` пройдёт (все реплики готовы)
4. **Проверка доступности базы данных**: `performConsistencyCheck` запустит `pg_isready`, чтобы убедиться, что PostgreSQL принимает соединения
5. **Финальный переход**: как только обе проверки пройдут, Database перейдёт в фазу `Ready`

**Ожидаемая последовательность логов:**
```
INFO    Handling Verifying phase    {"database": "consistency-failure-test"}
INFO    All replicas ready, checking consistency    {"replicas": 3}
INFO    Database accessibility check passed    {"endpoint": "consistency-failure-test.default.svc.cluster.local:5432"}
INFO    STATE TRANSITION: Verifying -> Ready    {"database": "consistency-failure-test"}
```

**Если реплики ещё не готовы:**
```
INFO    Consistency check failed, retrying    {"error": "not all replicas ready: 2/3"}
```

**Очистка:**
```bash
kubectl delete database consistency-failure-test
```

## Очистка

Удалите тестовые ресурсы, созданные во время этой лабораторной:

```bash
# Delete restore test resources
kubectl delete restore test-restore --ignore-not-found=true
kubectl delete restore test-restore-fail-db --ignore-not-found=true
kubectl delete restore test-restore-fail-backup --ignore-not-found=true

# Delete backup test resources
kubectl delete backup test-backup --ignore-not-found=true
kubectl delete backup restore-test-backup --ignore-not-found=true

# Delete database test resources
kubectl delete database test-db --ignore-not-found=true
kubectl delete database restore-test-db --ignore-not-found=true
kubectl delete database rolling-update-test --ignore-not-found=true
kubectl delete database consistency-test --ignore-not-found=true
```

## Итоги лабораторной

В этой лабораторной вы:
- Создали пакет резервного копирования с интеграцией `pg_dump`
- Сгенерировали каркас API Restore с помощью kubebuilder
- Реализовали контроллер Restore
- Добавили паттерны обработки скользящих обновлений
- Реализовали проверки согласованности данных

## Ключевые уроки

1. **Используйте kubebuilder для генерации каркаса новых API** — `kubebuilder create api` для Restore
2. **Разделяйте ответственность с помощью пакетов** — `internal/backup/` и `internal/restore/`
3. **Скользящие обновления обрабатываются StatefulSet** — контроллер просто обновляет spec
4. **Ждите завершения обновлений** — используйте паттерн `wait.PollImmediate`
5. **Согласованность данных специфична для приложения** — реализуйте проверки для вашей базы данных
6. **Координируйте несколько ресурсов** — Restore зависит и от Database, и от Backup

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Backup Implementation](../solutions/backup.go) — полная функциональность резервного копирования с `pg_dump`
- [Restore Implementation](../solutions/restore.go) — полная функциональность восстановления с `psql`
- [Rolling Update](../solutions/rolling-update.go) — обработка скользящих обновлений с логикой ожидания

### Использование решений

```bash
# Copy backup package
mkdir -p internal/backup
cp path/to/solutions/backup.go internal/backup/

# Copy restore package  
mkdir -p internal/restore
cp path/to/solutions/restore.go internal/restore/

# Reference rolling-update.go for Database controller enhancements
```

## Дальнейшие шаги

Теперь давайте создадим финальный проект!

**Навигация:** [← Предыдущая лабораторная: Композиция операторов](lab-02-operator-composition.md) | [Связанный урок](../lessons/03-stateful-applications.md) | [Следующая лабораторная: Финальный проект →](lab-04-final-project.md)
