# Артефакты и память проекта

## Главная идея

История чата удобна для разговора, но не является надёжной памятью проекта. Долгосрочная память должна быть структурированной, версионируемой и пригодной для выборочной загрузки.

## Предлагаемая структура

```text
research/
├── README.md
├── research-plan.md
├── questions.md
├── assumptions.md
├── workstreams/
├── investigations/
├── evidence/
├── experiments/
├── evaluations/
├── findings/
├── decisions/
├── synthesis/
├── context-packs/
└── archive/
```

Код продукта может жить отдельно от research artifacts, но связывается commit SHA и artifact IDs.

## Уровни знания

Важно не смешивать:

- **observation** — непосредственно увиденный факт;
- **evidence** — источник или измерение, поддерживающее observation;
- **interpretation** — объяснение наблюдений;
- **hypothesis** — проверяемое предположение;
- **finding** — вывод, прошедший review в заданной области;
- **decision** — выбранное действие с основаниями и последствиями;
- **assumption** — принятое временно без достаточного подтверждения.

Если тип не указан, последующий агент склонен превращать гипотезу в факт.

## Investigation artifact

Обязательные поля:

```yaml
id: INV-017
question: ...
scope: ...
method: ...
sources: []
repositories_examined: []
code_inspected: []
experiments_performed: []
observations: []
results: []
failed_approaches: []
limitations: []
confidence: low|medium|high
open_questions: []
recommendations: []
artifacts: []
produced_by:
  task_id: ...
  attempt_id: ...
  model: ...
review:
  status: pending|accepted|rejected
  reviewer_attempt_id: ...
```

## Evidence provenance

Для каждого источника сохраняются:

- стабильный URL или repository/commit/path;
- дата получения;
- фрагмент или координаты релевантного места;
- content hash для скачанного материала;
- способ извлечения;
- ограничения лицензии/цитирования;
- связь с observations.

Web-страница без даты доступа и commit-less ссылка на меняющийся код — слабая provenance.

## Отрицательные результаты

Неудачный эксперимент — ценный artifact, если указаны условия и причина. Он предотвращает повторение той же ветки другими агентами.

Записываются:

- что пытались сделать;
- почему ожидали успех;
- точная конфигурация;
- наблюдаемый результат;
- возможные объяснения;
- при каких изменениях стоит повторить.

## Decision records

Архитектурное решение оформляется коротким ADR:

```text
Context
Decision
Evidence
Alternatives considered
Consequences
Confidence
Revisit triggers
```

`Revisit triggers` особенно важны для долгого исследования: решение может быть правильным при текущих данных, но должно пересматриваться при появлении нового benchmark или ограничения.

## Synthesis

Synthesis не копирует все findings. Он создаёт карту:

- подтверждённое знание;
- противоречия;
- изменившиеся assumptions;
- решения;
- незакрытые риски;
- новые RQ;
- изменения DAG;
- рекомендуемый следующий цикл.

Каждое утверждение ссылается на artifact IDs.

## Context packs

Перед запуском worker получает минимальный пакет:

```text
goal excerpt
task definition + version
relevant decisions
relevant findings/evidence
input artifact manifest
tool and permission policy
output schema
acceptance criteria
```

Context pack имеет manifest и hash. Это позволяет воспроизвести попытку и не тащить в prompt весь проект.

## Что хранить в Git, а что нет

В Git:

- Markdown/YAML/JSON с планами и знаниями;
- небольшие reproducible scripts;
- конфигурации экспериментов;
- ссылки и manifests;
- результаты разумного размера.

В object storage:

- большие корпуса;
- архивы репозиториев;
- длинные логи;
- бинарные модели;
- видео/изображения;
- результаты, которые часто регенерируются.

В БД runtime:

- очереди, leases, timers;
- оперативные статусы;
- reservations;
- краткоживущая телеметрия.

## Garbage collection

Автоматическая очистка не удаляет evidence, принятые findings и decision inputs. Временные worktrees и сырые логи удаляются только после retention window и проверки, что их content hash/manifest сохранён.
