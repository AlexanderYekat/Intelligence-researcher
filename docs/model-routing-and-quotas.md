# Маршрутизация моделей и управление квотами

## Модели описываются способностями

Ядро не должно зависеть от названий Astra, Claude или GLM. Профиль исполнителя включает:

- tier рассуждения;
- качество coding/tool use;
- специализации;
- контекст и ограничения вывода;
- latency;
- денежную модель;
- доступные интерфейсы;
- требования к изоляции;
- историческую успешность на классах задач.

Названия конкретных моделей находятся в конфигурации deployment.

## Пример tier policy

| Tier | Назначение | Примеры задач |
|---|---|---|
| S | максимальный интеллект | program planning, архитектура, synthesis, adversarial/final review |
| A | сложная профессиональная работа | локальные планы, трудный код, глубокий research |
| B | стандартное исполнение | реализация, анализ репозитория, исправления |
| C | массовые формализованные операции | extraction, классификация, документация, простые тесты |

Планирование и review обычно получают больший tier, чем механическая реализация. Но это policy default, а не догма: benchmark может показать иное.

## Router

Router сначала отбрасывает несовместимых исполнителей, затем ранжирует оставшихся:

```text
route_score =
    predicted_success_probability × task_value
  + model_gain_for_task
  + quota_expiry_value
  - monetary_cost
  - quota_opportunity_cost
  - expected_latency
  - retry_risk
```

`model_gain_for_task` — ожидаемый прирост качества именно на данном типе работы, а не общий рейтинг модели.

Источники решения:

- declared capabilities;
- benchmark suite;
- история attempts;
- текущая доступность;
- пользовательские политики;
- независимость reviewer от автора.

## Автоматическая эскалация

Эскалация срабатывает по классифицированной причине:

- `insufficient_reasoning` → следующий tier;
- `missing_capability` → другой adapter/model;
- `context_overflow` → decomposition или другой context strategy;
- `tool_failure` → retry того же worker после восстановления;
- `bad_evidence` → revision с жёсткими критериями либо другой researcher;
- `ambiguous_goal` → planner/human, а не более дорогая генерация.

Эскалация ограничена бюджетом и не должна превращаться в бесконечную воронку.

## Quota pools

Лимит принадлежит не обязательно одной модели. Примеры:

- общий пятичасовой пул нескольких моделей;
- отдельный недельный пул high-tier модели;
- общий баланс API;
- локальный GPU pool;
- concurrency limit CLI;
- дневной лимит внешнего search API.

Пример состояния:

```yaml
id: openai-codex-weekly
provider: openai
kind: subscription_window
capacity_estimate: 100
remaining_estimate: 68
reset_at: 2026-09-18T07:03:00+05:00
confidence: medium
shared_by:
  - astra-xhigh
  - codex-work
source: cli_status
last_observed_at: 2026-09-17T23:10:00+05:00
```

Поскольку не все провайдеры дают официальный `remaining/reset_at` API, Broker должен различать:

- точные данные API;
- данные CLI/status page;
- response headers;
- распознанное сообщение об исчерпании;
- прогноз по собственной истории;
- ручной ввод.

Каждое значение имеет confidence и срок годности. Неизвестность нельзя выдавать за точный процент.

## Reservations

Перед запуском задача резервирует приблизительную квоту. После завершения reservation сверяется с фактическим расходом. Это предотвращает одновременный запуск множества workers, каждый из которых считает остаток свободным.

## Ожидание reset

При quota error adapter возвращает нормализованный результат:

```yaml
classification: quota_exhausted
quota_pool: openai-codex-5h
available_at: 2026-09-18T07:03:00+05:00
confidence: high
resume_strategy: retry_from_checkpoint
```

Workflow создаёт durable timer, освобождает worker и продолжает другие runnable-задачи.

## Quota harvesting

Если до reset осталось мало времени, а дорогая квота существенно не использована, её предельная стоимость приближается к нулю. Scheduler может выбрать задачи из opportunistic backlog.

Кандидат должен быть:

- полезным при любом исходе;
- неблокирующим срочный путь;
- подходящим сильной модели;
- ограниченным по времени;
- сохраняющим reusable artifact;
- безопасным для прерывания или повторения.

Пример utility:

```text
harvest_utility =
    expected_project_value
  × model_quality_gain
  × success_probability
  × artifact_reusability
  / expected_quota_cost
```

Подходящие задачи:

- red-team/adversarial review архитектуры;
- независимый synthesis;
- поиск скрытых допущений;
- альтернативный дизайн;
- анализ второстепенных кандидатов;
- подготовка benchmark или evaluation corpus.

Неподходящие задачи:

- генерация бессмысленного кода ради расхода;
- изменение production без review;
- задачи, создающие большой будущий maintenance cost;
- действия, нарушающие правила провайдера;
- работа без артефакта и критерия пользы.

## Anticipatory scheduling

Scheduler может заранее видеть, что скоро освободится дорогой tier, и подготовить context pack, источники и рабочее место дешёвыми workers. Тогда после reset сильная модель тратит квоту на рассуждение, а не на механический сбор данных.

## Метрики

- acceptance rate по model/task class;
- качество после независимого review;
- число escalation steps;
- стоимость принятого результата;
- wasted quota и unused expiring quota;
- latency на critical path;
- доля задач, повторённых из-за инфраструктуры;
- calibration точности quota estimate.
