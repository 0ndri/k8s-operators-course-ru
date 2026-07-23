---
layout: default
title: "Lab 02.4: First Operator"
nav_order: 14
parent: "Модуль 2: Введение в операторы"
grand_parent: Модули
mermaid: true
---

# Лабораторная 2.4: Создание оператора Hello World

**Связанный урок:** [Урок 2.4: Ваш первый оператор](../lessons/04-first-operator.md)  
**Навигация:** [← Предыдущая лабораторная: Среда разработки](lab-03-dev-environment.md) | [Обзор модуля](../README.md)

## Цели

- Создать свой первый полноценный оператор
- Понять структуру проекта оператора
- Запустить оператор локально
- Создавать пользовательскими ресурсами и управлять ими
- Наблюдать за согласованием в действии

## Предварительные требования

- Полноценная среда разработки из [Лабораторной 2.3](lab-03-dev-environment.md)
- Запущенный кластер kind
- Понимание CRD из [Модуля 1](../../module-01/README.md)

## Упражнение 1: инициализация проекта

### Задача 1.1: создайте каталог проекта

```bash
# Create project directory
mkdir -p ~/hello-world-operator
cd ~/hello-world-operator

# Initialize git (optional but recommended)
git init
```

### Задача 1.2: инициализируйте проект Kubebuilder

```bash
# Initialize kubebuilder project
kubebuilder init --domain example.com --repo github.com/example/hello-world-operator
```

**Обратите внимание:**
- Какие файлы были созданы?
- Какова структура проекта?

### Задача 1.3: проверьте структуру проекта

```bash
# List files
ls -la

# Check main.go
head -20 main.go

# Check Makefile
head -30 Makefile
```

## Упражнение 2: создание API

### Задача 2.1: создайте API HelloWorld

```bash
# Create API
kubebuilder create api --group hello --version v1 --kind HelloWorld
```

При запросах:
- Create Resource [y/n]: **y**
- Create Controller [y/n]: **y**

### Задача 2.2: изучите сгенерированные файлы

```bash
# Check API types
cat api/v1/helloworld_types.go

# Check controller
cat internal/controller/helloworld_controller.go
```

## Упражнение 3: определение типов API

### Задача 3.1: отредактируйте типы API

Отредактируйте `api/v1/helloworld_types.go`:

```go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// HelloWorldSpec defines the desired state of HelloWorld
type HelloWorldSpec struct {
    // Message is the message to display
    // +kubebuilder:validation:Required
    Message string `json:"message"`
    
    // Count is the number of times to display the message
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=100
    Count int32 `json:"count,omitempty"`
}

// HelloWorldStatus defines the observed state of HelloWorld
type HelloWorldStatus struct {
    // Phase represents the current phase
    Phase string `json:"phase,omitempty"`
    
    // ConfigMapCreated indicates if the ConfigMap was created
    ConfigMapCreated bool `json:"configMapCreated,omitempty"`
    
    // LastUpdated is when the status was last updated
    LastUpdated *metav1.Time `json:"lastUpdated,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Message",type="string",JSONPath=".spec.message"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"

// HelloWorld is the Schema for the helloworlds API
type HelloWorld struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   HelloWorldSpec   `json:"spec,omitempty"`
    Status HelloWorldStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// HelloWorldList contains a list of HelloWorld
type HelloWorldList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []HelloWorld `json:"items"`
}

func init() {
    SchemeBuilder.Register(&HelloWorld{}, &HelloWorldList{})
}
```

### Задача 3.2: сгенерируйте код

```bash
# Generate code
make generate

# Generate manifests
make manifests
```

### Задача 3.3: проверьте CRD

```bash
# Check CRD was generated
ls -la config/crd/bases/

# Examine CRD
cat config/crd/bases/hello.example.com_helloworlds.yaml | head -50
```

## Упражнение 4: реализация контроллера

### Задача 4.1: отредактируйте контроллер

Отредактируйте `internal/controller/helloworld_controller.go`:

```go
package controller

import (
    "context"
    "fmt"
    "time"
    
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/log"
    
    hellov1 "github.com/example/hello-world-operator/api/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// HelloWorldReconciler reconciles a HelloWorld object
type HelloWorldReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=hello.example.com,resources=helloworlds,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=hello.example.com,resources=helloworlds/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=hello.example.com,resources=helloworlds/finalizers,verbs=update
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete

// Reconcile is part of the main kubernetes reconciliation loop
func (r *HelloWorldReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    
    logger.Info("Reconciling HelloWorld", "name", req.NamespacedName)
    
    // Fetch the HelloWorld instance
    helloWorld := &hellov1.HelloWorld{}
    if err := r.Get(ctx, req.NamespacedName, helloWorld); err != nil {
        if errors.IsNotFound(err) {
            // Object not found, return
            logger.Info("HelloWorld not found, ignoring", "name", req.NamespacedName)
            return ctrl.Result{}, nil
        }
        // Error reading the object
        logger.Error(err, "Failed to get HelloWorld")
        return ctrl.Result{}, err
    }
    
    // Define the ConfigMap
    configMapName := helloWorld.Name + "-config"
    configMap := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      configMapName,
            Namespace: helloWorld.Namespace,
        },
        Data: map[string]string{
            "message": helloWorld.Spec.Message,
            "count":   fmt.Sprintf("%d", helloWorld.Spec.Count),
        },
    }
    
    // Set owner reference
    if err := ctrl.SetControllerReference(helloWorld, configMap, r.Scheme); err != nil {
        logger.Error(err, "Failed to set controller reference")
        return ctrl.Result{}, err
    }
    
    // Check if ConfigMap already exists
    existingConfigMap := &corev1.ConfigMap{}
    err := r.Get(ctx, client.ObjectKey{
        Name:      configMap.Name,
        Namespace: configMap.Namespace,
    }, existingConfigMap)
    
    if err != nil && errors.IsNotFound(err) {
        // ConfigMap doesn't exist, create it
        logger.Info("Creating ConfigMap", "name", configMap.Name)
        if err := r.Create(ctx, configMap); err != nil {
            logger.Error(err, "Failed to create ConfigMap")
            return ctrl.Result{}, err
        }
    } else if err != nil {
        logger.Error(err, "Failed to get ConfigMap")
        return ctrl.Result{}, err
    } else {
        // ConfigMap exists, update it if needed
        if existingConfigMap.Data["message"] != configMap.Data["message"] ||
           existingConfigMap.Data["count"] != configMap.Data["count"] {
            logger.Info("Updating ConfigMap", "name", configMap.Name)
            existingConfigMap.Data = configMap.Data
            if err := r.Update(ctx, existingConfigMap); err != nil {
                logger.Error(err, "Failed to update ConfigMap")
                return ctrl.Result{}, err
            }
        }
    }
    
    // Update status
    now := metav1.Now()
    helloWorld.Status.Phase = "Ready"
    helloWorld.Status.ConfigMapCreated = true
    helloWorld.Status.LastUpdated = &now
    
    if err := r.Status().Update(ctx, helloWorld); err != nil {
        logger.Error(err, "Failed to update status")
        return ctrl.Result{}, err
    }
    
    logger.Info("Successfully reconciled HelloWorld", "name", req.NamespacedName)
    return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *HelloWorldReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&hellov1.HelloWorld{}).
        Complete(r)
}
```

### Задача 4.2: перегенерируйте манифесты

```bash
# Regenerate RBAC (controller has new permissions)
make manifests
```

## Упражнение 5: установка и запуск оператора

### Задача 5.1: установите CRD

```bash
# Install CRD to cluster
make install

# Verify CRD was created
kubectl get crd helloworlds.hello.example.com

# Examine CRD
kubectl get crd helloworlds.hello.example.com -o yaml | head -30
```

### Задача 5.2: запустите оператор локально

В одном терминале запустите оператор:

```bash
# Run operator (connects to kind cluster)
make run
```

**Обратите внимание:**
- Оператор запускается
- Логи показывают, что он готов
- Он отслеживает ресурсы HelloWorld

### Задача 5.3: создайте ресурс HelloWorld

В другом терминале создайте HelloWorld:

```bash
# Create HelloWorld resource
cat <<EOF | kubectl apply -f -
apiVersion: hello.example.com/v1
kind: HelloWorld
metadata:
  name: hello-example
spec:
  message: "Hello from my first operator!"
  count: 5
EOF
```

### Задача 5.4: наблюдайте за согласованием

Посмотрите, что происходит:

```bash
# Check HelloWorld resource
kubectl get helloworld hello-example

# Get detailed view
kubectl get helloworld hello-example -o yaml

# Check ConfigMap was created
kubectl get configmap hello-example-config

# View ConfigMap data
kubectl get configmap hello-example-config -o jsonpath='{.data}'

# Watch operator logs (in the terminal running make run)
# You should see reconciliation logs
```

## Упражнение 6: тестирование обновлений

### Задача 6.1: обновите HelloWorld

```bash
# Update the message
kubectl patch helloworld hello-example --type merge -p '{"spec":{"message":"Updated message!"}}'

# Watch operator logs
# Check ConfigMap was updated
kubectl get configmap hello-example-config -o jsonpath='{.data.message}'
```

### Задача 6.2: проверьте обновления статуса

```bash
# Check status was updated
kubectl get helloworld hello-example -o jsonpath='{.status}'
```

## Упражнение 7: тестирование удаления

### Задача 7.1: удалите HelloWorld

```bash
# Delete HelloWorld
kubectl delete helloworld hello-example

# Check ConfigMap (should be deleted due to owner reference)
kubectl get configmap hello-example-config
```

**Ожидается:** ConfigMap должен быть автоматически удалён (ссылка-владелец из [Модуля 1](../../module-01/lessons/03-controller-pattern.md))

## Упражнение 8: создание нескольких ресурсов

### Задача 8.1: создайте несколько HelloWorld

```bash
# Create multiple HelloWorld resources
cat <<EOF | kubectl apply -f -
apiVersion: hello.example.com/v1
kind: HelloWorld
metadata:
  name: hello-1
spec:
  message: "First hello"
  count: 3
---
apiVersion: hello.example.com/v1
kind: HelloWorld
metadata:
  name: hello-2
spec:
  message: "Second hello"
  count: 7
EOF
```

### Задача 8.2: проверьте все ресурсы

```bash
# List all HelloWorlds
kubectl get helloworlds

# Check all ConfigMaps
kubectl get configmaps | grep hello
```

## Очистка

```bash
# Delete all HelloWorld resources
kubectl delete helloworlds --all

# Uninstall CRD
make uninstall

# Stop operator (Ctrl+C in the terminal running make run)
```

## Итоги лабораторной

В этой лабораторной вы:
- Создали полноценный проект оператора
- Определили типы пользовательского ресурса
- Реализовали логику согласования
- Запустили оператор локально
- Создавали пользовательскими ресурсами и управляли ими
- Наблюдали за согласованием в действии
- Протестировали обновления и удаления

## Ключевые уроки

1. Kubebuilder генерирует каркас полноценных проектов операторов
2. Вы определяете типы API (spec и status)
3. Вы реализуете функцию Reconcile
4. Оператор следует паттерну согласования из Модуля 1
5. Ссылки-владельцы управляют жизненным циклом ресурсов
6. Обновления статуса отражают фактическое состояние
7. Операторы запускаются локально, но подключаются к кластеру

## Поздравляем!

Вы создали свой первый оператор! Это демонстрирует:
- ✅ Создание CRD и управление ими
- ✅ Реализацию контроллера
- ✅ Паттерн согласования
- ✅ Создание и обновление ресурсов
- ✅ Управление статусом
- ✅ Ссылки-владельцы

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [main.go](../solutions/hello-world-operator-main.go) — полная точка входа оператора
- [Контроллер](../solutions/hello-world-controller.go) — полная реализация контроллера
- [Типы API](../solutions/hello-world-types.go) — полные определения типов API

## Дальнейшие шаги

В Модуле 3 вы научитесь создавать более совершенные контроллеры с продвинутыми паттернами!

**Навигация:** [← Предыдущая лабораторная: Среда разработки](lab-03-dev-environment.md) | [Связанный урок](../lessons/04-first-operator.md) | [Обзор модуля](../README.md)
