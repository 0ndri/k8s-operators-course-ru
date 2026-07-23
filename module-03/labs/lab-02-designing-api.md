---
layout: default
title: "Lab 03.2: Designing Api"
nav_order: 12
parent: "Модуль 3: Создание кастомных контроллеров"
grand_parent: Модули
mermaid: true
---

# Лабораторная 3.2: Проектирование API для оператора базы данных

**Связанный урок:** [Урок 3.2: Проектирование вашего API](../lessons/02-designing-api.md)  
**Навигация:** [← Предыдущая лабораторная: Controller Runtime](lab-01-controller-runtime.md) | [Обзор модуля](../README.md) | [Следующая лабораторная: Логика согласования →](lab-03-reconciliation-logic.md)

## Цели

- Спроектировать API для оператора базы данных PostgreSQL
- Использовать маркеры kubebuilder для валидации
- Сгенерировать CRD с корректной схемой
- Протестировать валидацию API

## Предварительные требования

- Завершение [Модуля 2](../../module-02/README.md)
- Понимание принципов проектирования API
- Установленный kubebuilder

## Упражнение 1: инициализация проекта оператора базы данных

### Задача 1.1: создайте проект

```bash
# Create new project
mkdir -p ~/postgres-operator
cd ~/postgres-operator

# Initialize kubebuilder project
kubebuilder init --domain example.com --repo github.com/example/postgres-operator
```

### Задача 1.2: создайте API Database

```bash
# Create Database API
kubebuilder create api --group database --version v1 --kind Database

# When prompted:
# Create Resource [y/n]: y
# Create Controller [y/n]: y
```

## Упражнение 2: проектирование Spec базы данных

### Задача 2.1: определите DatabaseSpec

Отредактируйте `api/v1/database_types.go`:

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    corev1 "k8s.io/api/core/v1"
)

// DatabaseSpec defines the desired state of Database
type DatabaseSpec struct {
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
}

// StorageSpec defines storage configuration
type StorageSpec struct {
    // Size is the storage size (e.g., "10Gi")
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:Pattern=`^[0-9]+(Gi|Mi)$`
    Size string `json:"size"`
    
    // StorageClass is the storage class to use
    StorageClass string `json:"storageClass,omitempty"`
}
```

### Задача 2.2: определите DatabaseStatus

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
}
```

### Задача 2.3: завершите тип Database

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Replicas",type="integer",JSONPath=".spec.replicas"
// +kubebuilder:printcolumn:name="Ready",type="boolean",JSONPath=".status.ready"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// Database is the Schema for the databases API
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// DatabaseList contains a list of Database
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Database `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Database{}, &DatabaseList{})
}
```

## Упражнение 3: генерация и проверка CRD

### Задача 3.1: сгенерируйте код

```bash
# Generate code
make generate

# Generate manifests
make manifests
```

### Задача 3.2: изучите сгенерированный CRD

```bash
# Check CRD was generated and verify validation rules
cat config/crd/bases/database.example.com_databases.yaml | head -100
```

**Вопросы:**
1. Присутствуют ли правила валидации?
2. Установлены ли значения по умолчанию?
3. Определены ли столбцы вывода?

## Упражнение 4: тестирование валидации API

### Задача 4.1: установите CRD

```bash
# Install CRD
make install

# Verify
kubectl get crd databases.database.example.com
```

### Задача 4.2: протестируйте корректный ресурс

```bash
# Create valid Database resource
cat <<EOF | kubectl apply -f -
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

# Verify it was created
kubectl get database test-db
```

### Задача 4.3: протестируйте некорректные ресурсы

```bash
# Test missing required field
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-db
spec:
  image: postgres:14
  # Missing databaseName
EOF

# Should fail validation

# Test invalid replica count
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-replicas
spec:
  image: postgres:14
  replicas: 20  # Exceeds maximum
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
EOF

# Should fail validation

# Test invalid storage size
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: invalid-storage
spec:
  image: postgres:14
  databaseName: mydb
  username: admin
  storage:
    size: invalid  # Doesn't match pattern
EOF

# Should fail validation
```

## Упражнение 5: тестирование столбцов вывода

### Задача 5.1: создайте несколько баз данных

```bash
# Create a few databases
kubectl apply -f - <<EOF
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: db1
spec:
  image: postgres:14
  replicas: 1
  databaseName: db1
  username: user1
  storage:
    size: 10Gi
---
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: db2
spec:
  image: postgres:13
  replicas: 2
  databaseName: db2
  username: user2
  storage:
    size: 20Gi
EOF
```

### Задача 5.2: проверьте столбцы вывода

```bash
# List databases - should show print columns
kubectl get databases

# Should show: NAME, PHASE, REPLICAS, READY, AGE
```

## Упражнение 6: тестирование значений по умолчанию

### Задача 6.1: создайте ресурс с минимальным Spec

```bash
# Create with only required fields
cat <<EOF | kubectl apply -f -
apiVersion: database.example.com/v1
kind: Database
metadata:
  name: minimal-db
spec:
  databaseName: mydb
  username: admin
  storage:
    size: 10Gi
  # image and replicas should use defaults
EOF
```

### Задача 6.2: проверьте значения по умолчанию

```bash
# Check if defaults were applied
kubectl get database minimal-db -o jsonpath='{.spec.image}'
kubectl get database minimal-db -o jsonpath='{.spec.replicas}'
```

## Очистка

```bash
# Delete test resources
kubectl delete databases --all

# Uninstall CRD
make uninstall
```

## Итоги лабораторной

В этой лабораторной вы:
- Спроектировали полноценный API для оператора базы данных
- Использовали маркеры kubebuilder для валидации
- Сгенерировали CRD с корректной схемой
- Протестировали правила валидации API
- Убедились, что столбцы вывода работают
- Протестировали значения по умолчанию

## Ключевые уроки

1. Дизайн API следует соглашениям Kubernetes
2. Spec содержит желаемое состояние, Status — фактическое
3. Маркеры валидации обеспечивают соблюдение ограничений
4. Столбцы вывода улучшают пользовательский опыт
5. Значения по умолчанию упрощают использование API
6. Правильное версионирование важно

## Решения

Дизайн API из этой лабораторной используется в полном решении оператора Database:
- [Database Types](../solutions/database-types.go) — полные определения типов API с маркерами валидации

## Дальнейшие шаги

Теперь, когда у вас есть хорошо спроектированный API, давайте реализуем логику согласования, чтобы он заработал!

**Навигация:** [← Предыдущая лабораторная: Controller Runtime](lab-01-controller-runtime.md) | [Связанный урок](../lessons/02-designing-api.md) | [Следующая лабораторная: Логика согласования →](lab-03-reconciliation-logic.md)
