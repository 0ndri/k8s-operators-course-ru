---
layout: default
title: "Lab 05.1: Admission Control"
nav_order: 11
parent: "Модуль 5: Вебхуки и контроль допуска"
grand_parent: Модули
mermaid: true
---

# Лабораторная 5.1: Исследование контроля допуска

**Связанный урок:** [Урок 5.1: Контроль допуска в Kubernetes](../lessons/01-admission-control.md)  
**Навигация:** [Обзор модуля](../README.md) | [Следующая лабораторная: Валидирующие вебхуки →](lab-02-validating-webhooks.md)

## Цели

- Изучить существующие контроллеры допуска
- Понять конфигурацию вебхука
- Протестировать эндпоинты вебхука
- Понять процесс контроля допуска

## Предварительные требования

- Завершение [Модуля 4](../../module-04/README.md)
- Запущенный кластер kind
- Понимание концепций контроля допуска

## Упражнение 1: изучение встроенных контроллеров допуска

### Задача 1.1: перечислите контроллеры допуска

```bash
# Check API server admission plugins enabled
# Note: Modern Kubernetes clusters have many default admission controllers enabled
# automatically (NamespaceLifecycle, LimitRanger, ServiceAccount, ResourceQuota,
# MutatingAdmissionWebhook, ValidatingAdmissionWebhook, etc.)

# For kind cluster, check API server args for admission plugins
kubectl get pod -n kube-system -l component=kube-apiserver -o yaml | grep -A 10 "admission"

# Alternative: Check the kube-apiserver manifest directly (kind-specific)
docker exec kind-control-plane cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -i admission

# If no --enable-admission-plugins flag is shown, the cluster uses the default set
# See: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#which-plugins-are-enabled-by-default
```

### Задача 1.2: протестируйте контроль допуска ResourceQuota

```bash
# Create a namespace with quota
kubectl create namespace quota-test
kubectl create quota test-quota --namespace=quota-test --hard=cpu=1,memory=1Gi

# Try to create pod that exceeds quota
kubectl run test-pod --image=nginx:latest --namespace=quota-test --overrides='{"spec":{"containers":[{"name":"test-pod","image":"nginx:latest","resources":{"requests":{"cpu":"2","memory":"2Gi"}}}]}}'

# Should be rejected by ResourceQuota admission controller
```

## Упражнение 2: изучение конфигураций вебхуков

### Задача 2.1: перечислите конфигурации вебхуков

```bash
# List validating webhook configurations
kubectl get validatingwebhookconfigurations

# List mutating webhook configurations
kubectl get mutatingwebhookconfigurations

# Get details
kubectl get validatingwebhookconfiguration <name> -o yaml
```

### Задача 2.2: изучите структуру вебхука

Если у вас установлены какие-либо вебхуки (например, от cert-manager):

```bash
# Get webhook configuration
kubectl get validatingwebhookconfiguration -o yaml | head -50

# Examine:
# - Rules (when webhook is called)
# - Client config (how to reach webhook)
# - Failure policy
```

## Упражнение 3: понимание правил вебхука

### Задача 3.1: проанализируйте структуру правила

Создайте пример конфигурации вебхука, чтобы понять структуру:

```bash
# Create sample webhook config (won't work without service, but shows structure)
cat <<EOF | kubectl apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: example-webhook
webhooks:
- name: example.example.com
  rules:
  - apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    operations: ["CREATE", "UPDATE"]
  clientConfig:
    service:
      name: example-webhook-service
      namespace: default
      path: "/validate"
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
EOF

# Examine it
kubectl get validatingwebhookconfiguration example-webhook -o yaml

# Delete it (it won't work anyway)
kubectl delete validatingwebhookconfiguration example-webhook
```

## Упражнение 4: тестирование процесса допуска

### Задача 4.1: создайте ресурс и отследите процесс

```bash
# Create a pod with verbose output
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
EOF

# Watch events to see admission process
kubectl get events --sort-by='.lastTimestamp' | tail -20
```

### Задача 4.2: протестируйте сбой валидации

```bash
# Try to create invalid resource
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: invalid-pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        cpu: "invalid"  # Invalid value
EOF

# Observe validation error
# This is caught by schema validation, not webhook
```

## Упражнение 5: понимание мутирующих против валидирующих

### Задача 5.1: наблюдайте за встроенными мутациями

```bash
# Create a pod without namespace
cat <<EOF > /tmp/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-mutation
spec:
  containers:
  - name: nginx
    image: nginx:latest
EOF

# Apply it
kubectl apply -f /tmp/pod.yaml

# Check what was added (mutations)
kubectl get pod test-mutation -o yaml | grep -A 10 "metadata:"

# Built-in admission controllers may add:
# - Default service account
# - Security context defaults
# - etc.
```

## Упражнение 6: требования к сервису вебхука

### Задача 6.1: разберитесь в требованиях к сервису

Чтобы вебхук работал, нужны:

1. **Service** — для маршрутизации запросов к подам вебхука
2. **Certificate** — для TLS-соединения
3. **Webhook Configuration** — для регистрации вебхука
4. **Webhook Handler** — для обработки запросов

```bash
# Check if any webhook services exist
kubectl get services -A | grep webhook

# Check webhook pods
kubectl get pods -A | grep webhook
```

## Очистка

```bash
# Clean up test resources
kubectl delete pod test-pod test-mutation invalid-pod 2>/dev/null || true
kubectl delete namespace quota-test 2>/dev/null || true
rm -f /tmp/pod.yaml
```

## Итоги лабораторной

В этой лабораторной вы:
- Изучили встроенные контроллеры допуска
- Изучили конфигурации вебхуков
- Разобрались в правилах и структуре вебхука
- Отследили процесс допуска
- Наблюдали за мутациями
- Разобрались в требованиях к вебхукам

## Ключевые уроки

1. Контроль допуска перехватывает запросы к API
2. Мутирующие вебхуки запускаются до валидирующих
3. Конфигурации вебхуков определяют, когда вызываются вебхуки
4. Вебхукам нужны сервисы и сертификаты
5. Встроенные контроллеры допуска обеспечивают базовую функциональность
6. Пользовательские вебхуки расширяют валидацию/мутацию

## Дальнейшие шаги

Теперь давайте создадим ваш собственный валидирующий вебхук!

**Навигация:** [← Обзор модуля](../README.md) | [Связанный урок](../lessons/01-admission-control.md) | [Следующая лабораторная: Валидирующие вебхуки →](lab-02-validating-webhooks.md)
