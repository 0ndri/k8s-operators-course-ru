# Решения Модуля 4

Этот каталог содержит полные рабочие решения для лабораторных Модуля 4.

## Файлы

- [**conditions-helpers.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-04/solutions/conditions-helpers.go): вспомогательные функции для управления условиями
- [**finalizer-handler.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-04/solutions/finalizer-handler.go): полная реализация финализатора
- [**watch-setup.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-04/solutions/watch-setup.go): примеры настройки отслеживания
- [**state-machine-controller.go**](https://github.com/piyushjajoo/k8s-operators-course/blob/main/module-04/solutions/state-machine-controller.go): полное многофазное согласование с конечным автоматом

## Использование

Эти решения можно использовать как:
- Справочный материал при добавлении условий в ваш оператор
- Отправную точку для реализации финализатора
- Примеры паттернов отслеживания
- Шаблон для реализации согласования на основе конечного автомата

## Интеграция

Чтобы использовать эти решения:

1. Добавьте вспомогательные функции для условий в ваш контроллер
2. Интегрируйте обработчик финализатора в функцию Reconcile
3. Обновите SetupWithManager конфигурацией отслеживания
4. Обновите тип status вашего Database, добавив Conditions
5. **Для конечного автомата**: замените основную функцию Reconcile так, чтобы она вызывала `reconcileWithStateMachine`

## Конечный автомат (Лабораторная 4)

Реализация конечного автомата обеспечивает многофазное согласование со следующим потоком состояний:

```
Pending → Provisioning → Configuring → Deploying → Verifying → Ready
                                                              ↓
                                                           Failed (on error)
```

### Предварительные требования для конечного автомата

1. **Обновите типы API** — отредактируйте `api/v1/database_types.go` и обновите enum поля Phase:
   ```go
   // +kubebuilder:validation:Enum=Pending;Provisioning;Configuring;Deploying;Verifying;Ready;Failed
   Phase string `json:"phase,omitempty"`
   ```

2. **Перегенерируйте и переустановите CRD**:
   ```bash
   make manifests
   make install
   ```

3. **Обновите функцию Reconcile** — основная функция `Reconcile` должна вызывать `reconcileWithStateMachine(ctx, db)` 
   вместо прямого вызова функций согласования ресурсов.

> **Примечание:** если вы пропустите шаги 1–2, вы увидите ошибки валидации вроде:
> `phase: Unsupported value: "Provisioning": supported values: "Pending", "Creating", "Ready", "Failed"`
>
> Если вы пропустите шаг 3, вы увидите только переходы `Pending → Creating → Ready`.

## Примечания

- Это полные рабочие примеры
- Условия следуют стандартам Kubernetes
- Финализаторы аккуратно обрабатывают очистку
- Отслеживание настроено корректно
- Конечный автомат обрабатывает все фазы, включая восстановление после ошибок
- Готовы к вебхукам из Модуля 5
