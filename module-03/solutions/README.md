# Решения Модуля 3

Этот каталог содержит полные рабочие решения для лабораторных Модуля 3.

## Файлы

- [**database-types.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-03/solutions/database-types.go): полные определения типов API Database
- [**database-controller.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-03/solutions/database-controller.go): полная реализация контроллера Database

## Использование

Эти решения можно использовать как:
- Справочный материал при создании оператора PostgreSQL
- Отправную точку, если вы застряли
- Примеры продвинутых паттернов согласования

## Интеграция

Чтобы использовать эти решения:

1. Создайте новый проект kubebuilder: `kubebuilder init --domain database.example.com --repo github.com/example/postgres-operator`
2. Создайте API: `kubebuilder create api --group database --version v1 --kind Database`
3. Замените сгенерированные файлы этими решениями
4. Запустите `make generate` и `make manifests`
5. Установите CRD: `make install`
6. Запустите оператор: `make run`

## Примечания

- Это полные рабочие примеры
- Реализовано согласование StatefulSet и Service
- Ссылки-владельцы для каскадного удаления
- Обновления статуса на основе фактического состояния
- Готовы к доработкам из Модуля 4 (условия, финализаторы)
