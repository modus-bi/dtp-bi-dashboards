# dtp-bi-dashboards

Можно ли собрать BI-дашборды за 4 часа, если ты не аналитик? 
Эксперимент с MCP PostgreSQL и MCP Modus BI.

## Структура репозитория

### HTML-отчёты

| Файл | Описание |
|------|----------|
| `hypothesis_analysis_plan.html` | План проверки 15 гипотез о ДТП — категории, методы, метрики |
| `hypothesis_complete_results.html` | Итерация 1 — результаты статистической проверки гипотез |
| `iteration2_normalized_results.html` | Итерация 2 — нормализованные результаты с учётом confounding-факторов |
| `dashboard_recommendations.html` | Рекомендации по составу дашбордов на основе подтверждённых гипотез |

### `img/` — скриншоты и диаграммы

Скриншоты готовых дашбордов, Mermaid-схемы структуры данных и дизайн-макеты отчётов.

### `skills/` — AI Skills для BI-аналитики

Набор инструкций в формате [Agent Skills](https://agentskills.io) — открытом стандарте, совместимом с Claude Code, Cursor, VS Code, GitHub Copilot и другими AI-инструментами. Подробнее — в [skills/README.md](skills/README.md).

## Data Quality (DQ) — автоматизированный аудит качества данных

Полный цикл DQM через MCP и AI Skills: профилирование → правила → эталоны → дашборд.

### Скиллы для DQ

| Skill | Назначение |
|-------|-----------|
| [`dq-profiler`](skills/dq-profiler/SKILL.md) | Полный цикл: профилирование (полнота, уникальность, энтропия) → правила → эталоны → дашборд |
| [`dq-rules-catalog`](skills/dq-rules-catalog/SKILL.md) | Reference-каталог стандартных DQ-правил по типам данных (ИНН/КПП/email/phone/даты) |
| [`dq-snapshot-history`](skills/dq-snapshot-history/SKILL.md) | Архивация результатов аудита в `dq_history` для trend во времени |
| [`create-dq-dashboard`](skills/create-dq-dashboard/SKILL.md) | Генерация мульти-отчётного DQ-дашборда через MCP API ModusBI |
| [`verify-report`](skills/verify-report/SKILL.md) | Детальная верификация отчёта ModusBI (фильтры, layout, данные, стили) |
| [`add-navigation-panel`](skills/add-navigation-panel/SKILL.md) | Навигационная панель между отчётами дашборда |

### `dq_html_example/` — HTML-снапшоты DQ-отчётов

Эталонные HTML-версии DQ-дашборда — референс при генерации новых отчётов:
[A.1 Объект](dq_html_example/a1_object.html), [A.2 Полнота](dq_html_example/a2_completeness.html), [A.3 Уникальность](dq_html_example/a3_uniqueness.html), [A.4 Валидность](dq_html_example/a4_validity.html), [A.5 Энтропия](dq_html_example/a5_entropy.html), [A.6 Дубли](dq_html_example/a6_duplicates.html), [A.7 DQ Score](dq_html_example/a7_dq_score.html), [B Правила](dq_html_example/b_rules.html), [C Эталоны](dq_html_example/c_golden.html). Старт — [index.html](dq_html_example/index.html).

### `dq_bi_screenshots/` — скрины DQ-отчётов из BI

Готовые DQ-отчёты, собранные в ModusBI: объект, полнота, ключи, формат, энтропия, дубли, итоговая оценка, правила, эталоны.
