# Решения Модуля 1

Этот каталог содержит полные рабочие решения для лабораторных Модуля 1.

## Файлы

- [**website-crd.yaml**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-01/solutions/website-crd.yaml): полное определение CRD Website
- [**example-website.yaml**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-01/solutions/example-website.yaml): пример пользовательского ресурса Website

## Использование

Эти решения можно использовать как:
- Справочный материал при создании собственных CRD
- Отправную точку, если вы застряли
- Примеры лучших практик CRD

## Интеграция

Чтобы использовать эти решения:

1. Примените CRD: `kubectl apply -f website-crd.yaml`
2. Дождитесь установки CRD: `kubectl wait --for condition=established crd/websites.example.com`
3. Создайте Website: `kubectl apply -f example-website.yaml`
4. Проверьте: `kubectl get websites`

## Примечания

- Это полные рабочие примеры
- Они следуют лучшим практикам Kubernetes
- Правила валидации включены
- Подресурс status настроен корректно
