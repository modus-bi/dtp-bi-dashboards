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
