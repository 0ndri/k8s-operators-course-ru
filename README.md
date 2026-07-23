
# Курс «Разработка операторов Kubernetes»

> ⚠️ Данный проект является переводом оригинального курса [Piyush Jajoo](https://github.com/piyushjajoo), доступного по ссылке [k8s-operators-course](https://github.com/piyushjajoo/k8s-operators-course). Благодарим автора за проделанную работу!


## Обзор курса

Этот курс научит вас создавать операторы Kubernetes с нуля. Вы изучите основы архитектуры Kubernetes, паттерн контроллера (controller pattern) и научитесь использовать Kubebuilder для создания собственных операторов, управляющих сложными приложениями.

**Продолжительность:** 8 недель (всего 40–50 часов)  
**Уровень:** от среднего до продвинутого  
**Предварительные требования:** базовые знания Kubernetes, основы программирования на Go, понимание контейнеризации  
**Лицензия:** бесплатный проект с открытым исходным кодом — распространяется по [лицензии MIT](LICENSE)

## Структура курса

Курс состоит из 8 модулей, каждый из которых опирается на предыдущий:

1. **[Модуль 1: Глубокое погружение в архитектуру Kubernetes](module-01/README.md)**
   - [Компоненты управляющего слоя](module-01/lessons/01-control-plane.md)
   - [Механизмы API (API machinery)](module-01/lessons/02-api-machinery.md)
   - [Паттерн контроллера](module-01/lessons/03-controller-pattern.md)
   - [Пользовательские ресурсы (Custom Resources)](module-01/lessons/04-custom-resources.md)

2. **[Модуль 2: Введение в операторы](module-02/README.md)**
   - [Паттерн оператора](module-02/lessons/01-operator-pattern.md)
   - [Основы Kubebuilder](module-02/lessons/02-kubebuilder-fundamentals.md)
   - [Среда разработки](module-02/lessons/03-dev-environment.md)
   - [Ваш первый оператор](module-02/lessons/04-first-operator.md)

3. **[Модуль 3: Создание кастомных контроллеров](module-03/README.md)**
   - [Controller runtime](module-03/lessons/01-controller-runtime.md)
   - [Проектирование API](module-03/lessons/02-designing-api.md)
   - [Логика согласования (reconciliation)](module-03/lessons/03-reconciliation-logic.md)
   - [Операции клиента](module-03/lessons/04-client-go.md)

4. **[Модуль 4: Продвинутые паттерны согласования](module-04/README.md)**
   - [Условия и статус](module-04/lessons/01-conditions-status.md)
   - [Финализаторы и очистка](module-04/lessons/02-finalizers-cleanup.md)
   - [Отслеживание и индексирование](module-04/lessons/03-watching-indexing.md)
   - [Продвинутые паттерны](module-04/lessons/04-advanced-patterns.md)

5. **[Модуль 5: Вебхуки и контроль допуска](module-05/README.md)**
   - [Контроль допуска (admission control)](module-05/lessons/01-admission-control.md)
   - [Валидирующие вебхуки](module-05/lessons/02-validating-webhooks.md)
   - [Мутирующие вебхуки](module-05/lessons/03-mutating-webhooks.md)
   - [Развёртывание вебхуков](module-05/lessons/04-webhook-deployment.md)

6. **[Модуль 6: Тестирование и отладка](module-06/README.md)**
   - [Основы тестирования](module-06/lessons/01-testing-fundamentals.md)
   - [Модульное тестирование](module-06/lessons/02-unit-testing-envtest.md)
   - [Интеграционное тестирование](module-06/lessons/03-integration-testing.md)
   - [Отладка и наблюдаемость](module-06/lessons/04-debugging-observability.md)

7. **[Модуль 7: Подготовка к продакшену](module-07/README.md)**
   - [Упаковка и распространение](module-07/lessons/01-packaging-distribution.md)
   - [RBAC и безопасность](module-07/lessons/02-rbac-security.md)
   - [Высокая доступность](module-07/lessons/03-high-availability.md)
   - [Производительность и масштабируемость](module-07/lessons/04-performance-scalability.md)

8. **[Модуль 8: Продвинутые темы и практические паттерны](module-08/README.md)**
   - [Мультиарендность и изоляция пространств имён](module-08/lessons/01-multi-tenancy.md)
   - [Композиция операторов](module-08/lessons/02-operator-composition.md)
   - [Управление stateful-приложениями](module-08/lessons/03-stateful-applications.md)
   - [Практические паттерны и лучшие практики](module-08/lessons/04-real-world-patterns.md)
