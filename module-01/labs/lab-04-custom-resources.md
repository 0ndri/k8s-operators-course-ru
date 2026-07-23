---
layout: default
title: "Lab 01.4: Custom Resources"
nav_order: 14
parent: "Модуль 1: Архитектура Kubernetes"
grand_parent: Модули
mermaid: true
---

# Лабораторная 1.4: Создание вашего первого CRD

**Связанный урок:** [Урок 1.4: Пользовательские ресурсы](../lessons/04-custom-resources.md)  
**Навигация:** [← Предыдущая лабораторная: Паттерн контроллера](lab-03-controller-pattern.md) | [Обзор модуля](../README.md)

## Цели

- Создать определение пользовательского ресурса (CRD)
- Создавать пользовательскими ресурсами и управлять ими
- Понять валидацию CRD
- Поработать с подресурсами status
- Понять, когда использовать CRD

## Предварительные требования

- Запущенный кластер kind
- Настроенный kubectl

## Упражнение 1: создание простого CRD

### Задача 1.1: определите CRD Website

```bash
# Create a CRD for managing websites
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              url:
                type: string
                pattern: '^https?://'
                description: The website URL
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                description: Number of replicas
              environment:
                type: string
                enum: [development, staging, production]
                default: development
            required:
            - url
            - replicas
          status:
            type: object
            properties:
              phase:
                type: string
                enum: [Pending, Running, Failed]
              readyReplicas:
                type: integer
              lastUpdated:
                type: string
                format: date-time
  scope: Namespaced
  names:
    plural: websites
    singular: website
    kind: Website
    shortNames:
    - ws
EOF

# Verify CRD was created
kubectl get crd websites.example.com

# Check API discovery
kubectl api-resources | grep websites
```

### Задача 1.2: проверьте эндпоинт API

```bash
# Check the API endpoint is available
kubectl get --raw /apis/example.com/v1

# Get the CRD definition
kubectl get crd websites.example.com -o yaml | head -50
```

## Упражнение 2: создание пользовательских ресурсов

### Задача 2.1: создайте корректный ресурс Website

```bash
# Create a website resource
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: my-blog
spec:
  url: https://example.com/blog
  replicas: 3
  environment: production
EOF

# Verify it was created
kubectl get websites
kubectl get website my-blog
kubectl get ws my-blog  # Using short name

# View the full resource
kubectl get website my-blog -o yaml
```

### Задача 2.2: создайте несколько сайтов

```bash
# Create more websites
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: my-shop
spec:
  url: https://example.com/shop
  replicas: 5
  environment: production
---
apiVersion: example.com/v1
kind: Website
metadata:
  name: dev-site
spec:
  url: http://dev.example.com
  replicas: 1
  environment: development
EOF

# List all websites
kubectl get websites

# Get specific website
kubectl get website my-shop -o yaml
```

## Упражнение 3: проверка валидации

### Задача 3.1: проверьте обязательные поля

```bash
# Try to create website without required field
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: invalid-website
spec:
  url: https://example.com
  # Missing replicas field
EOF
```

**Ожидаемый результат:** ошибка валидации об отсутствующем обязательном поле.

### Задача 3.2: проверьте валидацию шаблона URL

```bash
# Try invalid URL (doesn't match pattern)
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: invalid-url
spec:
  url: not-a-valid-url
  replicas: 2
EOF
```

**Ожидаемый результат:** ошибка валидации о шаблоне URL.

### Задача 3.3: проверьте валидацию диапазона реплик

```bash
# Try replicas below minimum
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: too-few-replicas
spec:
  url: https://example.com
  replicas: 0
EOF
```

**Ожидаемый результат:** ошибка валидации о минимальном значении.

```bash
# Try replicas above maximum
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: too-many-replicas
spec:
  url: https://example.com
  replicas: 20
EOF
```

**Ожидаемый результат:** ошибка валидации о максимальном значении.

### Задача 3.4: проверьте валидацию enum

```bash
# Try invalid environment value
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: invalid-env
spec:
  url: https://example.com
  replicas: 2
  environment: invalid-env
EOF
```

**Ожидаемый результат:** ошибка валидации о значении enum.

### Задача 3.5: проверьте значения по умолчанию

```bash
# Create website without environment (should use default)
cat <<EOF | kubectl apply -f -
apiVersion: example.com/v1
kind: Website
metadata:
  name: default-env
spec:
  url: https://example.com
  replicas: 2
EOF

# Check the default was applied
kubectl get website default-env -o jsonpath='{.spec.environment}'
echo
```

**Ожидаемый результат:** environment должно быть «development» (значение по умолчанию).

## Упражнение 4: обновление пользовательских ресурсов

### Задача 4.1: обновите spec

```bash
# Update the website
kubectl patch website my-blog --type merge -p '{"spec":{"replicas":5}}'

# Verify the update
kubectl get website my-blog -o jsonpath='{.spec.replicas}'
echo

# Update URL
kubectl patch website my-blog --type merge -p '{"spec":{"url":"https://newurl.com"}}'

# Verify
kubectl get website my-blog -o jsonpath='{.spec.url}'
echo
```

### Задача 4.2: обновите через YAML

```bash
# Get current resource
kubectl get website my-shop -o yaml > /tmp/website.yaml

# Edit the file (or use sed)
sed -i '' 's/replicas: 5/replicas: 7/' /tmp/website.yaml

# Apply the update
kubectl apply -f /tmp/website.yaml

# Verify
kubectl get website my-shop -o jsonpath='{.spec.replicas}'
echo
```

## Упражнение 5: подресурс status

### Задача 5.1: изучите поле status

```bash
# Get website with status
kubectl get website my-blog -o yaml | grep -A 10 status

# The status field exists but is empty (no controller to update it)
# In a real operator, the controller would update this
```

### Задача 5.2: разберитесь в spec и status

```bash
# Compare spec and status
echo "=== SPEC (Desired State) ==="
kubectl get website my-blog -o jsonpath='{.spec}' | jq '.'

echo -e "\n=== STATUS (Actual State) ==="
kubectl get website my-blog -o jsonpath='{.status}' | jq '.'

# Spec is what the user wants
# Status is what actually exists (updated by controller)
```

## Упражнение 6: сравнение CRD и ConfigMap

### Задача 6.1: создайте эквивалентный ConfigMap

```bash
# Create a ConfigMap with similar data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: website-config
data:
  url: "https://example.com"
  replicas: "3"
  environment: "production"
EOF

# Compare the two approaches
echo "=== Custom Resource ==="
kubectl get website my-blog -o yaml | head -20

echo -e "\n=== ConfigMap ==="
kubectl get configmap website-config -o yaml
```

**Ключевые различия:**
1. У CRD есть структурированная схема и валидация
2. ConfigMap — это просто пары «ключ-значение»
3. У CRD есть семантика API (можно отслеживать, есть resourceVersion)
4. У CRD может быть подресурс status

### Задача 6.2: попробуйте некорректные данные в ConfigMap

```bash
# ConfigMap accepts any data (no validation)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: invalid-config
data:
  url: "not-a-url"
  replicas: "not-a-number"
  environment: "invalid-env"
EOF

# This works! No validation
kubectl get configmap invalid-config

# But our CRD would reject this
```

## Упражнение 7: изучение деталей CRD

### Задача 7.1: изучите схему CRD

```bash
# Get the full CRD definition
kubectl get crd websites.example.com -o yaml > /tmp/crd.yaml

# View the schema
kubectl get crd websites.example.com -o jsonpath='{.spec.versions[0].schema}' | jq '.'

# View validation rules
kubectl get crd websites.example.com -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties}' | jq '.'
```

### Задача 7.2: обнаружение API

```bash
# Discover the API
kubectl get --raw /apis/example.com/v1 | jq '.'

# See available resources
kubectl get --raw /apis/example.com/v1 | jq '.resources[].name'

# Get a specific website via API
kubectl get --raw /apis/example.com/v1/namespaces/default/websites/my-blog | jq '.'
```

## Упражнение 8: удаление и очистка

### Задача 8.1: удалите пользовательские ресурсы

```bash
# Delete individual websites
kubectl delete website my-blog
kubectl delete website my-shop

# Delete all websites
kubectl delete websites --all

# Verify they're gone
kubectl get websites
```

### Задача 8.2: удалите CRD

```bash
# Delete the CRD
kubectl delete crd websites.example.com

# Verify it's gone
kubectl get crd websites.example.com

# Try to create a website (should fail)
kubectl create website test --url=https://test.com --replicas=2
```

**Примечание:** удаление CRD также удаляет все пользовательские ресурсы этого типа!

## Очистка

```bash
# Clean up any remaining resources
kubectl delete websites --all 2>/dev/null
kubectl delete crd websites.example.com 2>/dev/null
kubectl delete configmap website-config invalid-config 2>/dev/null
rm -f /tmp/website.yaml /tmp/crd.yaml
```

## Итоги лабораторной

В этой лабораторной вы:
- Создали определение пользовательского ресурса (CRD)
- Создавали пользовательскими ресурсами и управляли ими
- Проверили правила валидации CRD
- Сравнили CRD с ConfigMap
- Разобрались в разделении spec и status
- Изучили схему CRD и обнаружение API

## Ключевые уроки

1. CRD расширяют Kubernetes ресурсами, специфичными для предметной области
2. CRD обеспечивают валидацию схемы (в отличие от ConfigMap)
3. У CRD есть семантика API (watch, resourceVersion и т. д.)
4. Spec описывает желаемое состояние, status — фактическое
5. Валидация происходит на уровне API до сохранения
6. CRD — это основа для создания операторов

## Когда использовать CRD

**Используйте CRD, когда:**
- Нужны структурированные данные с валидацией
- Нужна семантика API (watch и т. д.)
- Вы создаёте оператор
- Нужен подресурс status

**Используйте ConfigMap, когда:**
- Простая конфигурация «ключ-значение»
- Валидация не нужна
- Семантика API не требуется

## Решения

Полные рабочие решения для этой лабораторной доступны в [каталоге решений](../solutions/):
- [Website CRD](../solutions/website-crd.yaml) — полное определение CRD
- [Example Website](../solutions/example-website.yaml) — пример пользовательского ресурса

**Навигация:** [← Предыдущая лабораторная: Паттерн контроллера](lab-03-controller-pattern.md) | [Связанный урок](../lessons/04-custom-resources.md) | [Обзор модуля](../README.md)
