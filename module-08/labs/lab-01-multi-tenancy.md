---
layout: default
title: "Lab 08.1: Multi Tenancy"
nav_order: 11
parent: "Модуль 8: Продвинутые темы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 8.1: Создание мультиарендного оператора

**Связанный урок:** [Урок 8.1: Мультиарендность и изоляция пространств имён](../lessons/01-multi-tenancy.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: Композиция операторов →](lab-02-operator-composition.md)

## Цели

- Сгенерировать каркас нового API с областью действия на кластер с помощью kubebuilder
- Сохранить существующий контроллер Database области действия на пространство имён
- Реализовать изоляцию пространств имён
- Обрабатывать квоты ресурсов
- Протестировать мультиарендные сценарии

## Предварительные требования

- Завершение [Модуля 7](../../module-07/README.md)
- Готовый оператор Database
- Понимание пространств имён и RBAC

## Обзор

В этой лабораторной вы создадите **новый** API с областью действия на кластер под названием `ClusterDatabase` наряду с существующим API `Database` области действия на пространство имён. Такой подход позволяет:

1. **Сохранить существующий контроллер Database** — изменения не нужны
2. **Изучить концепции области действия на кластер** — на выделенном API
3. **Сравнить оба подхода** — рядом в одном проекте

Ключевое отличие:
- `Database` (существующий): область действия на пространство имён, управляет базами данных внутри одного пространства имён
- `ClusterDatabase` (новый): область действия на кластер, управляет базами данных в любом пространстве имён

## Упражнение 1: генерация каркаса API с областью действия на кластер с помощью Kubebuilder

### Задача 1.1: создайте новый API

Используйте kubebuilder для генерации каркаса нового API ClusterDatabase:

```bash
# Navigate to your operator project
cd ~/postgres-operator

# Scaffold new API with cluster scope
kubebuilder create api \
  --group database \
  --version v1 \
  --kind ClusterDatabase \
  --resource --controller

# When prompted:
# Create Resource [y/n]: y
# Create Controller [y/n]: y
```

Это создаёт:
- `api/v1/clusterdatabase_types.go` — определения типов API
- `internal/controller/clusterdatabase_controller.go` — каркас контроллера

### Задача 1.2: настройте область действия на кластер

Отредактируйте `api/v1/clusterdatabase_types.go`, добавив маркер области действия на кластер:

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Cluster
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Namespace",type="string",JSONPath=".spec.targetNamespace"
// +kubebuilder:printcolumn:name="Tenant",type="string",JSONPath=".spec.tenant"
// +kubebuilder:printcolumn:name="Ready",type="boolean",JSONPath=".status.ready"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// ClusterDatabase is the Schema for the clusterdatabases API
// It is cluster-scoped and manages databases across namespaces
type ClusterDatabase struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ClusterDatabaseSpec   `json:"spec,omitempty"`
    Status ClusterDatabaseStatus `json:"status,omitempty"`
}
```

Ключевой маркер — `// +kubebuilder:resource:scope=Cluster`.

### Задача 1.3: определите Spec ClusterDatabase

Обновите spec в `api/v1/clusterdatabase_types.go`:

```go
// ClusterDatabaseSpec defines the desired state of ClusterDatabase
type ClusterDatabaseSpec struct {
    // Image is the PostgreSQL image to use
    // +kubebuilder:validation:Required
    // +kubebuilder:default="postgres:14"
    Image string `json:"image"`

    // Replicas is the number of database replicas
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas *int32 `json:"replicas,omitempty"`

    // Storage is the storage configuration
    Storage StorageSpec `json:"storage"`

    // Resources are the resource requirements
    Resources corev1.ResourceRequirements `json:"resources,omitempty"`

    // DatabaseName is the name of the database to create
    // +kubebuilder:validation:Required
    DatabaseName string `json:"databaseName"`

    // Username is the database user
    // +kubebuilder:validation:Required
    Username string `json:"username"`

    // TargetNamespace is where resources will be created
    // Required for cluster-scoped resources to know where to deploy
    // +kubebuilder:validation:Required
    TargetNamespace string `json:"targetNamespace"`

    // Tenant identifies which tenant owns this database
    // +optional
    Tenant string `json:"tenant,omitempty"`
}
```

Примечание: вы можете переиспользовать тип `StorageSpec` из вашего существующего API Database.

### Задача 1.4: определите Status ClusterDatabase

```go
// ClusterDatabaseStatus defines the observed state of ClusterDatabase
type ClusterDatabaseStatus struct {
    // Phase is the current phase
    // +kubebuilder:validation:Enum=Pending;Creating;Ready;Failed
    Phase string `json:"phase,omitempty"`

    // Ready indicates if the database is ready
    Ready bool `json:"ready,omitempty"`

    // Endpoint is the database endpoint
    Endpoint string `json:"endpoint,omitempty"`

    // SecretName is the name of the Secret containing credentials
    SecretName string `json:"secretName,omitempty"`

    // TargetNamespace shows where resources were created
    TargetNamespace string `json:"targetNamespace,omitempty"`

    // Conditions represent the latest observations
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}
```

### Задача 1.5: сгенерируйте и примените CRD

```bash
# Generate code and CRD manifests
make generate
make manifests

# Verify the CRD was generated with cluster scope
cat config/crd/bases/database.example.com_clusterdatabases.yaml | grep "scope:"
# Should output: scope: Cluster

# Install CRDs
make install

# Verify both CRDs exist
kubectl get crd | grep database.example.com
# Should show:
# clusterdatabases.database.example.com   (new, Cluster-scoped)
# databases.database.example.com          (existing, Namespaced)

# Check the scope
kubectl get crd clusterdatabases.database.example.com -o jsonpath='{.spec.scope}'
# Should output: Cluster
```

### Ключевые отличия от Database:

| Аспект | Database (Namespaced) | ClusterDatabase (Cluster-Scoped) |
|--------|----------------------|----------------------------------|
| Маркер области действия | (нет или `scope=Namespaced`) | `+kubebuilder:resource:scope=Cluster` |
| Пространство имён | Неявное из ресурса | Явное поле `targetNamespace` |
| Доступ | В рамках одного пространства имён | Во всех пространствах имён |
| Сценарий использования | Ресурсы уровня команды | Управление уровня платформы |

## Упражнение 2: реализация контроллера ClusterDatabase

### Задача 2.1: скопируйте полную реализацию контроллера

Контроллер ClusterDatabase похож на ваш существующий контроллер Database, но с ключевыми отличиями для ресурсов области действия на кластер. Вместо написания с нуля скопируйте полную реализацию из файла решений:

```bash
# Copy the complete controller implementation
cp path/to/solutions/clusterdatabase-controller.go internal/controller/clusterdatabase_controller.go
```

Или, если предпочитаете набрать сами, скопируйте полный контроллер из:
**[solutions/clusterdatabase-controller.go](../solutions/clusterdatabase-controller.go)**

Полный контроллер включает:
- `Reconcile()` — основной цикл согласования
- `validateNamespace()` — проверяет существование целевого пространства имён
- `checkQuota()` — проверяет квоты ресурсов
- `reconcileSecret()` — создаёт Secret с учётными данными в целевом пространстве имён
- `reconcileStatefulSet()` — создаёт StatefulSet в целевом пространстве имён
- `reconcileService()` — создаёт Service в целевом пространстве имён
- `updateStatus()` — обновляет статус ClusterDatabase

### Задача 2.2: разберитесь в ключевых отличиях от контроллера Database

Вот ключевые отличия в контроллере ClusterDatabase:

**1. Поле целевого пространства имён:**
```go
// Database controller uses implicit namespace from the resource
namespace := db.Namespace

// ClusterDatabase controller uses explicit targetNamespace
namespace := db.Spec.TargetNamespace
```

**2. Нет OwnerReferences (используются метки):**
```go
// Database controller can use OwnerReferences
ctrl.SetControllerReference(db, statefulSet, r.Scheme)

// ClusterDatabase controller CANNOT - use labels instead
statefulSet.Labels["clusterdatabase"] = db.Name
statefulSet.Labels["tenant"] = db.Spec.Tenant
```

**3. Валидация пространства имён:**
```go
// ClusterDatabase must validate target namespace exists
func (r *ClusterDatabaseReconciler) validateNamespace(ctx context.Context, namespace string) error {
    ns := &corev1.Namespace{}
    if err := r.Get(ctx, client.ObjectKey{Name: namespace}, ns); err != nil {
        if errors.IsNotFound(err) {
            return fmt.Errorf("target namespace %s does not exist", namespace)
        }
        return err
    }
    return nil
}
```

### Задача 2.3: убедитесь, что контроллер зарегистрирован

Kubebuilder автоматически регистрирует контроллер в `cmd/main.go`. Убедитесь, что это выглядит так:

```go
// This should already be added by kubebuilder
if err = (&controller.ClusterDatabaseReconciler{
    Client: mgr.GetClient(),
    Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
    setupLog.Error(err, "unable to create controller", "controller", "ClusterDatabase")
    os.Exit(1)
}
```

### Задача 2.4: соберите и проверьте

```bash
# Ensure the code compiles
make build

# If there are any compilation errors, verify you copied the complete
# controller from the solutions file
```

## Упражнение 3: обработка квот ресурсов

Функция `checkQuota` уже включена в скопированный вами файл решений. Разберёмся, как она работает, и протестируем её.

### Задача 3.1: создайте квоту ресурсов

Создайте `config/samples/quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: database-quota
  namespace: tenant-1
spec:
  hard:
    # Limit ClusterDatabases targeting this namespace
    clusterdatabases.database.example.com: "5"
```

### Задача 3.2: разберитесь в проверке квот

Функция `checkQuota` в вашем контроллере (из решений) работает так:

```go
func (r *ClusterDatabaseReconciler) checkQuota(ctx context.Context, namespace string) error {
    quota := &corev1.ResourceQuota{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      "database-quota",
        Namespace: namespace,
    }, quota)

    if errors.IsNotFound(err) {
        // No quota, proceed
        return nil
    }
    if err != nil {
        return err
    }

    // Count ClusterDatabases targeting this namespace
    databases := &databasev1.ClusterDatabaseList{}
    if err := r.List(ctx, databases); err != nil {
        return err
    }

    var count int64
    for _, db := range databases.Items {
        if db.Spec.TargetNamespace == namespace {
            count++
        }
    }

    hard, exists := quota.Spec.Hard["clusterdatabases.database.example.com"]
    if !exists {
        return nil
    }

    if hard.Value() <= count {
        return fmt.Errorf("quota exceeded: %d/%d clusterdatabases", count, hard.Value())
    }

    return nil
}
```

Ключевые моменты:
- Проверяет, существует ли ResourceQuota в целевом пространстве имён
- Считает все ClusterDatabase, нацеленные на это пространство имён (список по всему кластеру, затем фильтр)
- Возвращает ошибку, если квота была бы превышена

## Упражнение 4: тестирование мультиарендных сценариев

> **Предварительные требования:** убедитесь, что вы завершили Упражнение 2 (скопировали полный контроллер из решений) и ваш код компилируется через `make build`.

### Задача 4.1: соберите и разверните оператор в кластер Kind

Поскольку операторы с вебхуками (из предыдущих модулей) требуют TLS-сертификатов и развёртывания в кластере, мы развернём в кластер kind:

```bash
# Verify code compiles
make build

# Generate code and manifests
make generate manifests

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

# Check logs
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

### Задача 4.2: создайте пространства имён арендаторов

В новом терминале (или в том же после развёртывания):

```bash
# Create namespaces for tenants
kubectl create namespace tenant-1
kubectl create namespace tenant-2

# Label namespaces for tenant identification
kubectl label namespace tenant-1 tenant=tenant-1
kubectl label namespace tenant-2 tenant=tenant-2
```

### Задача 4.3: создайте ClusterDatabase для разных арендаторов

```bash
# Create ClusterDatabase for tenant-1
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: ClusterDatabase
metadata:
  name: cdb-tenant-1-prod
spec:
  targetNamespace: tenant-1
  tenant: tenant-1
  image: postgres:14
  replicas: 1
  databaseName: proddb
  username: admin
  storage:
    size: "10Gi"
EOF

# Create ClusterDatabase for tenant-2
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: ClusterDatabase
metadata:
  name: cdb-tenant-2-prod
spec:
  targetNamespace: tenant-2
  tenant: tenant-2
  image: postgres:14
  replicas: 1
  databaseName: proddb
  username: admin
  storage:
    size: "10Gi"
EOF
```

### Задача 4.4: проверьте изоляцию

```bash
# List all ClusterDatabases (cluster-wide view)
kubectl get clusterdatabases

# Output shows all databases with their target namespaces:
# NAME               PHASE   NAMESPACE   TENANT     READY   AGE
# cdb-tenant-1-prod  Ready   tenant-1    tenant-1   true    1m
# cdb-tenant-2-prod  Ready   tenant-2    tenant-2   true    1m

# Verify resources are created in correct namespaces
kubectl get statefulsets -n tenant-1
kubectl get statefulsets -n tenant-2

# Filter by tenant using jsonpath
kubectl get clusterdatabases -o jsonpath='{range .items[?(@.spec.tenant=="tenant-1")]}{.metadata.name}{"\n"}{end}'
```

> **Ресурсы не создаются?** Проверьте логи оператора:
> ```bash
> kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager
> ```

### Задача 4.5: сравните с Database области действия на пространство имён

```bash
# You can still use the namespace-scoped Database in parallel
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: local-db
  namespace: tenant-1
spec:
  image: postgres:14
  replicas: 1
  databaseName: localdb
  username: user
  storage:
    size: "5Gi"
EOF

# List both types
kubectl get databases -n tenant-1    # Shows namespace-scoped
kubectl get clusterdatabases          # Shows cluster-scoped
```

## Упражнение 5: понимание ограничений владения

Файл решений уже реализует эти паттерны. Это упражнение объясняет концепции, чтобы вы понимали, что происходит.

### Задача 5.1: ограничения владельца области действия на кластер

Важно: ресурсы области действия на кластер **не могут** использовать `OwnerReferences`, чтобы владеть ресурсами области действия на пространство имён. Файл решений вместо этого использует метки.

Во вспомогательной функции `buildStatefulSet` устанавливаются метки для отслеживания владения:

```go
func (r *ClusterDatabaseReconciler) buildStatefulSet(db *databasev1.ClusterDatabase) *appsv1.StatefulSet {
    // ... replicas and image setup ...

    return &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Spec.TargetNamespace,
            Labels: map[string]string{
                // Use labels to track ownership instead of OwnerReferences
                "app.kubernetes.io/managed-by": "clusterdatabase-controller",
                "clusterdatabase":              db.Name,
                "tenant":                       db.Spec.Tenant,
            },
        },
        // ... spec ...
    }
}
```

Обратите внимание на ключевое отличие от контроллера Database области действия на пространство имён:

```go
// Database controller (namespace-scoped) - CAN use OwnerReferences:
ctrl.SetControllerReference(db, statefulSet, r.Scheme)  // ✓ Works

// ClusterDatabase controller (cluster-scoped) - CANNOT use OwnerReferences:
// ctrl.SetControllerReference(db, statefulSet, r.Scheme)  // ✗ Would fail
// Instead, we use labels and cleanup with finalizers
```

### Задача 5.2: очистка с помощью финализаторов

Поскольку мы не можем использовать OwnerReferences для автоматической сборки мусора, файл решений реализует финализаторы. Вот как они работают:

```go
const clusterDatabaseFinalizer = "database.example.com/clusterdatabase-finalizer"

func (r *ClusterDatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    db := &databasev1.ClusterDatabase{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Handle deletion
    if !db.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(db, clusterDatabaseFinalizer) {
            // Clean up managed resources
            if err := r.cleanupManagedResources(ctx, db); err != nil {
                return ctrl.Result{}, err
            }
            controllerutil.RemoveFinalizer(db, clusterDatabaseFinalizer)
            return ctrl.Result{}, r.Update(ctx, db)
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(db, clusterDatabaseFinalizer) {
        controllerutil.AddFinalizer(db, clusterDatabaseFinalizer)
        return ctrl.Result{}, r.Update(ctx, db)
    }

    // ... rest of reconciliation
}

func (r *ClusterDatabaseReconciler) cleanupManagedResources(ctx context.Context, db *databasev1.ClusterDatabase) error {
    logger := log.FromContext(ctx)
    namespace := db.Spec.TargetNamespace

    // Delete StatefulSet by name
    statefulSet := &appsv1.StatefulSet{}
    err := r.Get(ctx, client.ObjectKey{Name: db.Name, Namespace: namespace}, statefulSet)
    if err == nil {
        logger.Info("Deleting StatefulSet", "name", db.Name, "namespace", namespace)
        if err := r.Delete(ctx, statefulSet); err != nil && !errors.IsNotFound(err) {
            return err
        }
    } else if !errors.IsNotFound(err) {
        return err
    }

    // Delete Service by name
    service := &corev1.Service{}
    err = r.Get(ctx, client.ObjectKey{Name: db.Name, Namespace: namespace}, service)
    if err == nil {
        logger.Info("Deleting Service", "name", db.Name, "namespace", namespace)
        if err := r.Delete(ctx, service); err != nil && !errors.IsNotFound(err) {
            return err
        }
    } else if !errors.IsNotFound(err) {
        return err
    }

    // Delete Secret by name
    secret := &corev1.Secret{}
    secretName := r.secretName(db)
    err = r.Get(ctx, client.ObjectKey{Name: secretName, Namespace: namespace}, secret)
    if err == nil {
        logger.Info("Deleting Secret", "name", secretName, "namespace", namespace)
        if err := r.Delete(ctx, secret); err != nil && !errors.IsNotFound(err) {
            return err
        }
    } else if !errors.IsNotFound(err) {
        return err
    }

    return nil
}
```

### Задача 5.3: протестируйте поведение очистки

```bash
# Ensure tenant-1 namespace exists (from earlier)
kubectl get namespace tenant-1 || kubectl create namespace tenant-1

# Create a ClusterDatabase
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: ClusterDatabase
metadata:
  name: test-cleanup
spec:
  targetNamespace: tenant-1
  tenant: tenant-1
  image: postgres:14
  replicas: 1
  databaseName: testdb
  username: admin
  storage:
    size: "5Gi"
EOF

# Verify resources were created
kubectl get statefulsets -n tenant-1

# Watch operator logs in another terminal to see cleanup happening
# kubectl logs -n postgres-operator-system deployment/postgres-operator-controller-manager -f

# Delete the ClusterDatabase
kubectl delete clusterdatabase test-cleanup

# Verify resources were cleaned up by the finalizer
kubectl get statefulsets -n tenant-1
# The StatefulSet should be deleted
```

## Очистка

```bash
# Delete ClusterDatabases
kubectl delete clusterdatabases --all

# Delete test namespaces
kubectl delete namespace tenant-1 tenant-2

# (Optional) Undeploy operator
make undeploy
```

## Итоги лабораторной

В этой лабораторной вы:
- Сгенерировали каркас нового API области действия на кластер с помощью kubebuilder
- Сохранили существующий контроллер Database области действия на пространство имён
- Реализовали изоляцию пространств имён с помощью `targetNamespace`
- Добавили обработку квот ресурсов
- Протестировали мультиарендные сценарии
- Узнали об ограничениях владения при области действия на кластер

## Ключевые уроки

1. **Используйте kubebuilder для генерации каркаса новых API** — `kubebuilder create api` берёт на себя шаблонный код
2. **Используйте маркер `+kubebuilder:resource:scope=Cluster`** — делает CRD с областью действия на кластер
3. **Ресурсам области действия на кластер нужны явные поля пространства имён** — используйте `targetNamespace`
4. **Нельзя использовать OwnerReferences между областями действия** — вместо этого используйте метки и финализаторы
5. **Оба контроллера могут сосуществовать** — каждый управляет своим типом ресурса
6. **`make manifests` генерирует CRD** — не нужно писать YAML CRD вручную

## Сравнение: Database против ClusterDatabase

| Возможность | Database | ClusterDatabase |
|---------|----------|-----------------|
| Область действия | Namespaced | Cluster |
| Пространство имён | Неявное | Явное (`targetNamespace`) |
| OwnerReferences | Да | Нет (используются метки) |
| Очистка | Автоматическая (GC) | Ручная (финализаторы) |
| RBAC | На пространство имён | На весь кластер |
| Сценарий использования | Ресурсы команды | Управление платформой |

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [ClusterDatabase Types](../solutions/clusterdatabase-types.go) — полные определения типов API
- [ClusterDatabase Controller](../solutions/clusterdatabase-controller.go) — полная реализация контроллера
- [Multi-Tenant Controller](../solutions/multi-tenant-controller.go) — пример паттернов мультиарендности

## Дальнейшие шаги

Теперь давайте изучим композицию операторов!

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-multi-tenancy.md) | [Следующая лабораторная: Композиция операторов →](lab-02-operator-composition.md)
