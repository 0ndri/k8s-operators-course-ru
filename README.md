---
layout: default
title: Обзор курса
nav_order: 0
nav_exclude: true
---

# Курс «Разработка операторов Kubernetes»

Полноценный практический и бесплатный курс по созданию готовых к продакшену операторов Kubernetes с помощью Kubebuilder.

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

## Начало работы

### Предварительные требования

- Go 1.24+
- kubectl
- Docker или Podman
- kind v0.29+
- Kubebuilder 4.7+

### Настройка

1. **Склонируйте этот репозиторий:**

   ```bash
   git clone https://github.com/piyushjajoo/k8s-operators-course.git
   cd k8s-operators-course
   ```

2. **Настройте среду разработки:**

   ```bash
   ./scripts/setup-dev-environment.sh
   ```

3. **Создайте кластер kind:**

   ```bash
   ./scripts/setup-kind-cluster.sh
   ```

4. **Начните с [Модуля 1](module-01/README.md):**

   ```bash
   cd module-01
   cat README.md
   ```

## Подход к обучению

Этот курс делает упор на:

- **Практическое обучение:** каждая концепция демонстрируется через практические упражнения
- **Наглядность:** активное использование диаграмм Mermaid для архитектуры и процессов
- **Постепенное усложнение:** начинаем с простого и доходим до готовых к продакшену операторов
- **Реальные примеры:** вы создаёте настоящие операторы, которые можно использовать

## Ресурсы

- [Документация Kubebuilder](https://book.kubebuilder.io/)
- [Документация API Kubernetes](https://kubernetes.io/docs/reference/kubernetes-api/)
- [Паттерн оператора](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Полный код оператора Hello World](https://github.com/piyushjajoo/hello-world-operator), созданный по этому курсу, — можно использовать как справочный материал.
- [Полный код оператора Postgres](https://github.com/piyushjajoo/postgres-operator), созданный по этому курсу, — можно использовать как справочный материал.

## Участие в проекте

Мы приветствуем вклад и обратную связь! Вот как вы можете помочь улучшить этот курс:

### Сообщения об ошибках

Если вы нашли баги, опечатки или ошибки в материалах курса, пожалуйста, откройте issue в этом репозитории.

### Запрос новых тем

Есть идея новой концепции, темы или модуля, которые вы хотели бы видеть в курсе? Мы будем рады услышать вас!

**Чтобы предложить новую тему:**

1. **Откройте новый issue** в этом репозитории с меткой `enhancement` (если доступна) или используйте префикс заголовка `[Feature Request]`
2. **Укажите следующую информацию:**
   - **Название концепции/темы:** какую тему вы хотели бы видеть раскрытой?
   - **Описание:** краткое описание концепции и того, чем она будет полезна
   - **Предлагаемый модуль:** в какой модуль это лучше всего впишется? (или предложите новый модуль)
   - **Сценарий использования:** как это поможет учащимся создавать более качественные операторы?
   - **Приоритет:** это желательное дополнение или критический пробел в курсе?

3. **Пример формата:**
   ```
   [Feature Request] Operator SDK Comparison

   Description: Add a lesson comparing Kubebuilder with Operator SDK
   Suggested Module: Module 2 or new comparison module
   Use Case: Help learners understand when to choose which framework
   Priority: Nice-to-have
   ```

Мы рассматриваем все запросы и расставляем приоритеты на основе:
- Интереса сообщества и количества голосов
- Соответствия учебным целям курса
- Сложности и времени, необходимых для разработки
- Пробелов в текущем покрытии курса

**Примечание:** мы не можем гарантировать, что каждый запрос будет реализован, но ценим ваше мнение и рассмотрим все предложения!

## Лицензия

Этот курс **бесплатен и имеет открытый исходный код**, он распространяется по [лицензии MIT](LICENSE). Вы можете свободно:

- Использовать, распространять и изменять материалы курса
- Использовать их в личных или коммерческих целях
- Распространять и сублицензировать материалы

Единственное требование — сохранять оригинальное уведомление об авторских правах и текст лицензии. Полные условия см. в файле [LICENSE](LICENSE).

## Поделитесь своим проектом

Если вы прошли курс и создали оператор, мы будем рады его увидеть! Поделитесь своим проектом в LinkedIn и отметьте [Piyush Jajoo](https://www.linkedin.com/in/pjajoo). Я постараюсь в свободное время просмотреть ваш код и дать обратную связь. Пожалуйста, поставьте проекту ⭐, если он оказался полезным.

## Поддержка

По вопросам и для обсуждений, пожалуйста, откройте issue в этом репозитории.
