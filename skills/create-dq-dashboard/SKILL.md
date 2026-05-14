---
name: create-dq-dashboard
description: Создаёт мульти-отчётный Data Quality дашборд через MCP API ModusBI. Принимает таблицу dq_snapshots-результатов и генерирует 4-8 связанных отчётов с навигацией сверху, KPI, графиками, Pivot, картой и conditional formatting. Использовать когда нужен DQ-дашборд после прогона `/dq-profiler`. Универсален — работает на любых данных, соответствующих контракту dq_snapshots.
license: MIT
metadata:
    skill-author: ModusBI Team
    version: "1.0"
    domain: data-quality
    mcp-servers: postgres, modusbi, chrome-devtools
user-invocable: true
---

# Create DQ Dashboard: мульти-отчётный DQ-дашборд

## Overview

Создаёт **набор связанных отчётов** с навигационной панелью сверху, объединённый общими DQ-метриками. Каждый отчёт — глубокая детализация одного аспекта качества данных по DAMA DMBOK (Completeness, Validity, Uniqueness, Consistency, Timeliness).

**Что отличает от других `/create-*`:**
- Единственный скилл с **multi-report** архитектурой через `/add-navigation-panel`.
- Опирается на готовый контракт `dq_snapshots` (от `/dq-profiler`) — не считает метрики, только визуализирует.
- 5-dimensional DAMA color-blind safe Okabe-Ito палитра.
- Severity и Status как **независимые цветовые оси** (severity = бордюр, status = fill).

**Зависит от:**
- `/dq-profiler` — для генерации `dq_snapshots`.
- `/add-navigation-panel` — для нав-панели на каждом отчёте.
- `/verify-report` — для финальной верификации.

---

## When to Use

- Запуск после `/dq-profiler`: «Прогнал DQ-аудит, нужен дашборд».
- «Создай DQ-дашборд на таблице X».
- «Покажи DQ-метрики на BI».

**Не использовать если:**
- Метрик ещё нет в БД → сначала `/dq-profiler`.
- Нужен один simple-отчёт → `/create-simple-report`.
- Нужен KPI без DQ-контекста → `/create-kpi-dashboard`.

---

## Контракт входов

| Параметр | Тип | Обязателен | Описание |
|---|---|---|---|
| `snapshots_table` | text | ✓ | Таблица DQ-результатов (например, `mos.dq_snapshots`) |
| `datasource_id` | int | ✓ | ID datasource в ModusBI (через `datasource_operations(action="list")`) |
| `group_id` | int | ✓ | ID группы отчётов |
| `dashboard_name` | text | ✓ | Заголовок («DQ Audit · data.mos.ru») |
| `top_violations_table` | text | — | Опц., таблица с топ-нарушителями для drill-down раздела |
| `geo_join_query` | text | — | Опц., SQL-фрагмент для JOIN с region/lat/lon |
| `enable_reports` | list | — | `[overview, completeness, validity, trend]` (default MVP) или `*` для всех 8 |
| `dq_score_thresholds` | [int, int] | — | `[60, 80]` default — пороги жёлтого/зелёного |
| `locale` | text | — | `"ru"` (default) или `"en"` |

### Минимальный контракт `snapshots_table`

```sql
rule_id        text NOT NULL
rule_type      text NOT NULL    -- NOT_NULL/FORMAT/UNIQUENESS/FK/CONSISTENCY/FRESHNESS/RANGE/PARSEABLE/CROSS_FIELD/NEAR_DUPLICATE/ENTROPY
table_name     text NOT NULL
field_name     text             -- может быть NULL для cross-table правил
total_rows     bigint NOT NULL
violations     bigint NOT NULL
violation_pct  numeric          -- generated либо считается inline в SQL датасета
severity       text             -- CRITICAL/HIGH/MEDIUM/LOW (NULL → MEDIUM)
run_at         timestamptz      -- timestamp запуска для trend
```

Если каких-то колонок нет — Шаг 0 даёт подсказку миграции.

---

## Архитектура — multi-report с навигацией

### Список отчётов (генерируется по `enable_reports`)

| # | Slug | Каноническое имя | Аудитория | Decision-to-make | Когда создаётся | MVP по умолчанию |
|---|---|---|---|---|---|---|
| 1 | `dq-overview` | Обзор | C-level / CDO | Эскалировать или нет | всегда | ✅ |
| 2 | `dq-completeness` | Полнота | Data Engineer / DBA | Какой ETL чинить первым | всегда | ✅ |
| 3 | `dq-validity` | Валидность | Data Steward | Какие правила подтянуть | всегда | ✅ |
| 4 | `dq-trend` | Динамика | CDO / DQ-team | Регрессия или прогресс | если `COUNT(DISTINCT run_at) >= 2` | ✅ |
| 5 | `dq-uniqueness` | Уникальность | Data Steward / MDM | Что дедуплицировать | всегда | — |
| 6 | `dq-consistency` | Согласованность | Architect / DE | Какую FK чинить | всегда | — |
| 7 | `dq-violators` | Топ-нарушители | Steward + бизнес | Какие записи дочистить | если задан `top_violations_table` | — |
| 8 | `dq-geo` | География | Бизнес-аналитик | Какой регион — bottleneck | если есть geo-поля | — |

### Навигация — через `/add-navigation-panel`

После создания всех отчётов вызывается готовый скилл:

```
Создай навигационную панель TabsPanel сверху каждого из созданных отчётов:
  /report/{overview_id}     "Обзор"            активный=true для overview, иначе false
  /report/{completeness_id} "Полнота"          активный=true для completeness
  /report/{validity_id}     "Валидность"
  /report/{trend_id}        "Динамика"
  ...

Палитра (gradient blue по образцу 663-668):
  bgColor      = rgba(21,101,192,1)
  активная     = rgba(255,255,255,0.15) полупрозрачный белый
  неактивная   = rgba(255,255,255,0)    прозрачный
  текст        = rgba(255,255,255,1)
Layout: y=0, h=3, w=60, link_target="_self".
Сдвинь все остальные компоненты y+=3.
```

`/add-navigation-panel` сам выберет способ A (TabsPanel native) и реализует.

---

## Layout главного отчёта (Overview, 60-grid HD)

```
y=0  h=3  TabsPanel (через /add-navigation-panel)

y=3  h=2  HsSubheaderComponent
          "DQ Audit · {datasource} · {N rules} · last run {ts} · run #{N}"

y=5  h=10 ┌─ DQ Score (15w) ──┬─ 5 KPI dimensions (45w, 9w каждый) ─┐
          │ HsTotalsPanelV2   │ HsTotalsPanelV2 × 5                │
          │ value=dq_score    │ Compl/Valid/Uniq/Cons/Time         │
          │ 70px bold         │ 36px каждый, Okabe-Ito цвета       │
          │ цвет по threshold │                                    │
          └───────────────────┴────────────────────────────────────┘

y=15 h=12 ┌─ Severity donut (25w) ─┬─ Trend sparkline (35w) ────────┐
          │ HsPieAMchartsChart     │ HsLineAMchartsChart            │
          │ severity-палитра       │ только dq_score over run_at    │
          │ legend visible         │ smooth, без осей (sparkline)   │
          │                        │ если history<2 — placeholder   │
          └────────────────────────┴────────────────────────────────┘

y=27 h=15 ┌─ Top 10 правил по нарушениям (60w) ─────────────────────┐
          │ HsBarAMchartsChart, horizontal                          │
          │ category=rule_id, value=violations                      │
          │ severity-цвета через conditional_rule_add               │
          └─────────────────────────────────────────────────────────┘
```

**8 компонентов на главной** (включая TabsPanel + Subheader). C-level-friendly.

---

## Layout остальных отчётов (краткие схемы)

### `dq-completeness`
```
y=0  h=3  TabsPanel
y=3  h=8  ┌── KPI: Completeness Score (15w) ──┬── % полей с >50% NULL (15w) ──┬── Полей всего (15w) ──┬── Records audited (15w) ──┐
y=11 h=14 ┌── Pivot heatmap rule × table (60w) ──────────────────────────────────────────────────────────────────────────────────────┐
y=25 h=12 ┌── Bar tables ranked by completeness (30w) ────┬── Top 20 fields with NULL >50% (30w, table) ────┐
```

### `dq-validity`
```
y=0  h=3  TabsPanel
y=3  h=8  4 KPI: Validity Score · Format violations · Range outliers · Date parse fails
y=11 h=14 Bar by rule_type · Top invalid samples table (50/50)
y=25 h=12 HsTextBlock (примеры топ-5 невалидных значений + объяснение)
```

### `dq-trend`
```
y=0  h=3  TabsPanel
y=3  h=8  HsLineAMchartsChart — DQ Score 30/90/365 days
y=11 h=14 ┌── Top deteriorating rules (table) ──┬── Top improving rules (table) ──┐
y=25 h=12 HsRadarAMchartsChart — current run vs previous run
```

> Полные layouts остальных — см. `examples/`.

---

## Цветовые палитры

### 5 dimensions DAMA — Okabe-Ito (color-blind safe)
```
completeness  rgba(  0, 114, 178, 1)  blue
validity      rgba(  0, 158, 115, 1)  bluish green
uniqueness    rgba(240, 228,  66, 1)  yellow
consistency   rgba(213,  94,   0, 1)  vermillion
timeliness    rgba(204, 121, 167, 1)  reddish purple
```

### Severity vs Status — независимые оси цвета

**Severity** (приоритет правила) — через **бордюр**, не fill:
```
CRITICAL    border-left: 4px solid #b71c1c
HIGH        border-left: 3px solid #e65100
MEDIUM      border-left: 2px solid #fbc02d
LOW         no border
```

**Status** (PASS/WARN/FAIL результат) — через **fill**:
```
PASS    bg #c8e6c9, text #1b5e20
WARN    bg #fff9c4, text #f57f17
FAIL    bg #ffcdd2, text #c62828
```

### DQ Score thresholds (input parameter)
```
≥ thresholds[1] (default 80)         bg #2e7d32, text #fff
[thresholds[0], thresholds[1])       bg #f9a825, text #fff
< thresholds[0] (default 60)         bg #c62828, text #fff
```

---

## Component types для типов данных

| Что показываем | Компонент | Почему |
|---|---|---|
| Single big number (KPI, score) | `HsTotalsPanelV2` | Стандарт |
| Severity composition (4 категории) | `HsPieAMchartsChart` (donut) | <7 категорий → читается |
| Trend over time | `HsLineAMchartsChart` | Стандарт |
| Bar ranking (top-N) | `HsBarAMchartsChart` horizontal | Длинные имена правил |
| Pivot rule × table | `HsPivotChart` + conditional formatting | Двумерная агрегация |
| Hierarchical proportions | `HsTreeMap` | Вложенность |
| Сравнение текущий vs предыдущий run | `HsRadarAMchartsChart` | Только в `dq-trend` |
| Geo distribution | `HsMapLeafletChart` + choropleth | В `dq-geo` |
| Detailed table | `HsTableReactabularChart` + `conditional_rule_add` | Drill-down + цвет |
| Длинный текстовый Level 4 | `HsTextBlock` или `add_chart_with_conclusion` | Объяснение |
| Подзаголовки | `HsSubheaderComponent` | Иерархия |

> ⚠️ **`HsSpeedometerV2AMcharts` (gauge)** — раньше не использовали (Tufte/Few critique). С 2026-05-06 разрешён для **процентных DQ-метрик** (DQ Score, completeness %): шкала 0-100, 4 цветные зоны, полукруг (`grid.startAngle=180`, `grid.endAngle=360`). Документация ModusBI помечает «API creatable: нет», но через generic `component_crud(action="add", component_type="HsSpeedometerV2AMcharts", value_field="completeness_score", aggregation="max", ...)` создаётся корректно. Подтверждение — отчёт 744 idx=17. Для не-процентных метрик (счётчики, абсолютные числа) по-прежнему используем `HsTotalsPanelV2` + threshold colors.

---

## Workflow в 7 шагов

### Шаг 0. Probe реальной схемы (обязательный)

```python
# 0.1. Структура таблицы
mcp__postgres__query(sql=f"""
    SELECT column_name, data_type, is_nullable
    FROM information_schema.columns
    WHERE table_schema = '{schema}' AND table_name = '{table}'
""")
# Проверить наличие: rule_id, rule_type, table_name, field_name,
# total_rows, violations, severity, run_at

# 0.2. История?
mcp__postgres__query(sql=f"""
    SELECT COUNT(DISTINCT run_at) AS runs
    FROM {snapshots_table}
""")
# runs >= 2 → enable_reports.trend = True

# 0.3. Geo есть?
mcp__postgres__query(sql=f"""
    SELECT EXISTS(
        SELECT 1 FROM information_schema.columns
        WHERE table_schema = '{schema}' AND table_name = '{table}'
          AND column_name IN ('region', 'adm_area', 'district')
    )
""")
# Если есть — enable_reports.geo = True

# 0.4. Top violations?
# Передан пользователем top_violations_table → enable_reports.violators = True
```

При отсутствии обязательной колонки — прерываем с подсказкой:
```
ERROR: В таблице {snapshots_table} нет колонки 'severity'.
Добавь её через:
    ALTER TABLE {snapshots_table} ADD COLUMN severity text DEFAULT 'MEDIUM';
Или используй /dq-profiler v3.0+ который пишет severity автоматически.
```

### Шаг 1. Создание датасетов (inline SQL, без CREATE VIEW)

postgres-mcp в нашей конфигурации **readonly**. Все агрегаты — внутри SQL `dataset_create_from_sql`.

**`dq_main`** — для Bar, Pivot, Incident feed:
```python
result = mcp__modusbi__dataset_create_from_sql(
    sql_query=f"""
        SELECT
            rule_id,
            rule_type,
            table_name,
            field_name,
            total_rows,
            violations,
            ROUND(100.0 * violations / NULLIF(total_rows, 0), 2) AS violation_pct,
            CASE
                WHEN violations = 0 THEN 'PASS'
                WHEN 100.0 * violations / NULLIF(total_rows, 0) < 5 THEN 'WARNING'
                ELSE 'FAIL'
            END AS status,
            COALESCE(severity, 'MEDIUM') AS severity,
            CASE rule_type
                WHEN 'NOT_NULL' THEN 'completeness'
                WHEN 'FORMAT' THEN 'validity'
                WHEN 'RANGE' THEN 'validity'
                WHEN 'PARSEABLE' THEN 'validity'
                WHEN 'UNIQUENESS' THEN 'uniqueness'
                WHEN 'NEAR_DUPLICATE' THEN 'uniqueness'
                WHEN 'FK' THEN 'consistency'
                WHEN 'CROSS_FIELD' THEN 'consistency'
                WHEN 'CONSISTENCY' THEN 'consistency'
                WHEN 'FRESHNESS' THEN 'timeliness'
                ELSE 'other'
            END AS dimension
        FROM {snapshots_table}
        WHERE run_at = (SELECT MAX(run_at) FROM {snapshots_table})
        ORDER BY
            CASE COALESCE(severity, 'MEDIUM')
                WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2
                WHEN 'MEDIUM' THEN 3 WHEN 'LOW' THEN 4
            END,
            violations DESC
    """,
    datasource_id=datasource_id,
    name="dq_main",
    caption="DQ — последний запуск"
)
dq_main_id = result["dataset"]["id"]

# Алиасы — отдельным вызовом (dataset_create_from_sql не принимает field_aliases)
mcp__modusbi__dataset_operations(
    action="update",
    dataset_id=dq_main_id,
    field_aliases={
        "rule_id":       {"alias": "ID правила"},
        "violations":    {"alias": "Нарушений"},
        "violation_pct": {"alias": "%"},
        "severity":      {"alias": "Severity"},
        "status":        {"alias": "Статус"}
    }
)
```

**`dq_dimensions`** — для 5 KPI и Radar:
```sql
WITH base AS (
    SELECT rule_type, total_rows, violations,
        CASE rule_type
            WHEN 'NOT_NULL' THEN 'completeness'
            WHEN 'FORMAT' THEN 'validity'
            -- ... тот же mapping
        END AS dimension
    FROM <snapshots_table>
    WHERE run_at = (SELECT MAX(run_at) FROM <snapshots_table>)
)
SELECT dimension,
       COUNT(*) AS rules_count,
       -- 100.0 - 100.0*violations/total_rows уже даёт 0..100 (% пройденных) — внешнего ×100 не нужно
       ROUND(AVG(GREATEST(0, 100.0 - 100.0 * violations / NULLIF(total_rows, 0))), 1) AS dimension_score
FROM base
GROUP BY dimension
ORDER BY CASE dimension
    WHEN 'completeness' THEN 1 WHEN 'validity' THEN 2
    WHEN 'uniqueness' THEN 3 WHEN 'consistency' THEN 4
    WHEN 'timeliness' THEN 5 ELSE 6
END
```

**`dq_score_kpi`** — главная цифра + 5 dimension значений в одной строке (для удобной привязки KPI):
```sql
WITH per_dim AS ( /* SQL из dq_dimensions */ ),
weights(dim, w) AS (VALUES
    ('completeness', 0.30), ('validity', 0.25), ('uniqueness', 0.20),
    ('consistency',  0.15), ('timeliness', 0.10)
)
SELECT
    ROUND(SUM(p.dimension_score * w.w), 1) AS dq_score,
    MAX(CASE WHEN p.dimension='completeness' THEN p.dimension_score END) AS completeness,
    MAX(CASE WHEN p.dimension='validity'     THEN p.dimension_score END) AS validity,
    MAX(CASE WHEN p.dimension='uniqueness'   THEN p.dimension_score END) AS uniqueness,
    MAX(CASE WHEN p.dimension='consistency'  THEN p.dimension_score END) AS consistency,
    MAX(CASE WHEN p.dimension='timeliness'   THEN p.dimension_score END) AS timeliness
FROM per_dim p
JOIN weights w ON w.dim = p.dimension
```

**`dq_trend`** — только если `enable_reports.trend = True`:
```sql
SELECT DATE_TRUNC('day', run_at) AS day,
       ROUND(AVG(GREATEST(0, 100.0 - 100.0 * violations / NULLIF(total_rows, 0))), 1) AS dq_score
FROM <snapshots_table>
GROUP BY 1 ORDER BY 1
```

### Шаг 2. Создание N отчётов

```python
report_ids = {}
for slug, name in {
    "overview":      "Обзор",
    "completeness":  "Полнота",
    "validity":      "Валидность",
    "trend":         "Динамика",
    # ... остальные если активны
}.items():
    if slug in active_reports:
        r = mcp__modusbi__report_operations(
            action="create",
            name=f"{dashboard_name} · {name}",
            caption=f"{dashboard_name} · {name}",  # обязательный параметр в v2.0
            group_id=group_id
        )
        # Возврат: {"report": {"id":..., "name":..., "caption":..., "group_id":..., "group_name":...}}
        report_ids[slug] = r["report"]["id"]
```

### Шаг 3. Заполнение каждого отчёта через `batch_operations`

**ВАЖНО:** ключ компонента в `batch_operations` — `component_type`, не `type`.

```python
mcp__modusbi__batch_operations(
    action="components_add",
    report_id=report_ids["overview"],
    components=[
        {"component_type":"HsSubheaderComponent",
         "title":f"DQ Audit · {datasource_name}",
         "x":0, "y":0, "w":60, "h":2},
        {"component_type":"HsTotalsPanelV2", "title":"DQ Score",
         "value_field":"dq_score", "aggregation":"sum",
         "dataset_id":dq_score_id,
         "x":0, "y":2, "w":15, "h":10},
        # 5 KPI dimensions:
        {"component_type":"HsTotalsPanelV2", "title":"Completeness",
         "value_field":"completeness", "aggregation":"sum",
         "dataset_id":dq_score_id,
         "x":15, "y":2, "w":9, "h":10},
        # ... аналогично validity / uniqueness / consistency / timeliness
        # Severity donut:
        {"component_type":"HsPieAMchartsChart", "title":"Severity",
         "category_field":"severity", "value_field":"violations",
         "aggregation":"sum",
         "dataset_id":dq_main_id,
         "x":0, "y":12, "w":25, "h":12},
        # Trend sparkline:
        {"component_type":"HsLineAMchartsChart", "title":"DQ Score · 30 дней",
         "category_field":"day", "value_field":"dq_score",
         "aggregation":"avg",
         "dataset_id":dq_trend_id,
         "x":25, "y":12, "w":35, "h":12},
        # Top 10 правил:
        {"component_type":"HsBarAMchartsChart", "title":"Топ-10 нарушений",
         "category_field":"rule_id", "value_field":"violations",
         "aggregation":"sum",
         "dataset_id":dq_main_id,
         "x":0, "y":24, "w":60, "h":15},
    ]
)
```

Аналогичный batch — для каждого из остальных 3-7 отчётов.

### Шаг 4. Conditional formatting через `component_styling`

```python
# ⚠️ Severity цвета через conditional formatting НЕ работают через MCP API:
# `conditional_rule_add` принимает только NUMERIC value (`could not convert string to float`).
# Для severity-цветов нужен один из двух обходов:
#
# Обход A (рекомендуется): добавить numeric `severity_priority` в SQL `dq_main`:
#     CASE COALESCE(severity, 'MEDIUM')
#         WHEN 'CRITICAL' THEN 1 WHEN 'HIGH' THEN 2
#         WHEN 'MEDIUM' THEN 3 WHEN 'LOW' THEN 4
#     END AS severity_priority
# и применить conditional_rule_add на этой колонке (operator="=", value=1, color=red, ...).
#
# Обход B: вынести severity colors в CSS через `style_operations(action="update", css=...)`.
# Менее надёжно, требует знания структуры HTML портала.

# Status на табличных компонентах
for status, bg, fg in [
    ("PASS",    "#c8e6c9", "#1b5e20"),
    ("WARNING", "#fff9c4", "#f57f17"),
    ("FAIL",    "#ffcdd2", "#c62828"),
]:
    mcp__modusbi__component_styling(
        action="conditional_rule_add",
        report_id=report_ids["validity"],
        component_idx=validity_table_idx,
        config_overrides={
            "field":"status", "operator":"=", "value":status,
            "background":bg, "color":fg
        }
    )

# DQ Score thresholds (3 правила на главную KPI)
mcp__modusbi__component_styling(
    action="conditional_rule_add",
    report_id=report_ids["overview"], component_idx=dq_score_kpi_idx,
    config_overrides={"field":"dq_score","operator":">=","value":dq_score_thresholds[1],
                 "background":"#2e7d32","color":"#fff"}
)
mcp__modusbi__component_styling(
    action="conditional_rule_add",
    report_id=report_ids["overview"], component_idx=dq_score_kpi_idx,
    config_overrides={"field":"dq_score","operator":"<","value":dq_score_thresholds[0],
                 "background":"#c62828","color":"#fff"}
)
```

### Шаг 5. Навигационная панель — `/add-navigation-panel`

> **⚠️ Известная проблема MCP v2.0:**
> `tabs_button_add` через MCP-tool возвращает `Missing required parameters: ["title"]` — fastmcp не пробрасывает Annotated[str|None]=None параметр `title` в JSON-schema.
>
> **Workaround:** добавление кнопок через прямой вызов `component_operations` функции (а не MCP-tool) с указанием `title`. Это не блокирует skill — но требует одного дополнительного Python-скрипта при первом запуске. См. пример обхода в `examples/_add_tabs_buttons.py`.
>
> **При исправлении** в `app/server.py` (явное `title: str = ""` без Annotated default) — переход на чистый MCP-вызов будет тривиальным.

```
Используй /add-navigation-panel для каждого из созданных отчётов.

Кнопки в порядке (одинаковые на всех отчётах, активная подсвечивается на текущем):
1. /report/{overview_id}     "Обзор"
2. /report/{completeness_id} "Полнота"
3. /report/{validity_id}     "Валидность"
4. /report/{trend_id}        "Динамика"
(остальные если активны)

Палитра:
  bgColor      = rgba(21,101,192,1)
  активная     = rgba(255,255,255,0.15)
  неактивная   = rgba(255,255,255,0)
  текст        = rgba(255,255,255,1)

Position: y=0, h=3, w=60. Сдвиг остальных компонентов y+=3 на каждом отчёте.
Способ A (HsTabsPanel native).
```

### Шаг 6. Глобальные фильтры внутри каждого отчёта

3 фильтра одинаковых на каждом отчёте — через `component_filters`:

```python
# Сначала добавить HsFilterControlPanel
filter_panel_idx = ...  # после batch_operations

for slug, rid in report_ids.items():
    for value_field, alias in [
        ("severity",   "Severity"),
        ("dimension",  "Измерение"),
        ("table_name", "Таблица"),
    ]:
        mcp__modusbi__component_filters(
            action="filter_field_add",
            report_id=rid,
            component_idx=filter_panel_idx,
            value_field=value_field,    # параметр value_field, не field
            alias=alias,
            dataset_id=dq_main_id
        )
```

⚠️ **Cross-report фильтр НЕ работает** — `set_global_filter` действует только внутри одного report. Для каждого из N отчётов фильтры настраиваются независимо.

### Шаг 7. Визуальная верификация — `/verify-report`

После всех creations:

```
Используй /verify-report для каждого из созданных отчётов:
- {overview_id}: проверь что DQ Score рендерится цифрой (не placeholder),
  severity-цвета на месте, нет undefined, TabsPanel сверху.
- {completeness_id}: Pivot heatmap не пустой, KPI Completeness отрисован.
- {validity_id}: цвета status (PASS/WARN/FAIL) корректные.
- {trend_id}: линия не пустая (если history >= 2).
- (остальные если активны)
```

`/verify-report` использует chrome-devtools MCP для скриншотов и API-проверки.

---

## Output Format

После завершения работы скилл возвращает:

```markdown
# DQ Dashboard создан

## Дашборд: {dashboard_name}

### Созданные отчёты
| Slug | Название | Report ID | URL |
|---|---|---|---|
| overview | Обзор | 1234 | https://devcopy.modusbi.ru/report/1234 |
| ... | ... | ... | ... |

### Метрики (последний прогон)
- **DQ Score:** 68.2 / 100 (yellow)
- **Total rules:** 15
- **Total violations:** 312,420
- **Records audited:** 551,074
- **Datasets covered:** 4

### Что дальше
- Запусти cron `/dq-profiler` раз в сутки для накопления Trend
- Подключи alerts через инфраструктуру (вне MCP)
- Кастомизируй severity-цвета через `component_styling(conditional_rule_update, ...)`
```

---

## Граничные случаи

| Кейс | Поведение |
|---|---|
| `snapshots_table` не существует | Прерывание с подсказкой «Запустите `/dq-profiler`» |
| Нет колонки `severity` | Migrate-SQL подсказка `ALTER TABLE ... ADD COLUMN severity text DEFAULT 'MEDIUM'` |
| Только 1 правило | Скрываем Pivot и Bar, оставляем KPI + Subheader |
| 1 запуск (`run_at`) | Раздел Trend не создаётся, в TabsPanel кнопки нет |
| 1 `table_name` | Pivot становится бессмысленным — заменяем на rule × dimension |
| Нет geo-полей | Раздел Geo не создаётся |
| MCP в readonly | Ошибка с подсказкой `--readwrite` |
| `datasource_id` неверен | `datasource_operations(action="list")` и список доступных |
| chrome-devtools-mcp недоступен | Шаг 7 пропускается с warning |
| Все правила = PASS (Score=100) | Дашборд зелёный + HsTextBlock «Все правила прошли. Это не значит, что данные идеальны — может, не хватает правил» |
| `top_violations_table` не передан | Раздел `dq-violators` не создаётся |

---

## Что НЕ входит (вне MCP API)

| Желаемое | Почему не входит | Куда переносим |
|---|---|---|
| Auto-refresh `report_options.refresh_interval` | Нет в MCP API | Cron на инфраструктуре |
| Notifications/alerts на FAIL | Нет в MCP API | Внешний alert-канал (Slack/email) |
| `set_global_filter` cross-report | Только внутри одного report | URL-параметры (фронтенд) |
| `pivot_link_set` на Bar | Работает только на `HsPivotChart` | Если нужен drill — заменить Bar на Pivot |
| `security_operations(filter=...)` | Реально `(dataset_id, enabled, rls_query)` | RLS на dataset для PII-таблиц |

---

## Best Practices

1. **Перед запуском** проверь `mcp__modusbi__instance_operations(action="get_active")` — портал указан верно.
2. **Для одного датасорса** не создавай >1 DQ-дашборда параллельно — будут конфликты по `name`.
3. **`top_violations_table`** должна иметь индекс по `record_id` или `global_id` — для быстрого drill-down.
4. **Для cron** — после создания дашборда настрой `/dq-profiler` на регулярный запуск; trend заполнится сам.
5. **Локализация** — параметр `locale` влияет только на subheader/tabs/aliases. Технические имена полей — английский всегда.

---

## ⚠️ Known MCP bugs — реальные грабли (актуально на 2026-05-05)

> Полный разбор: `doc/habr/article2/mcp_real_bugs.md`. Здесь — короткий чек-лист для skill-исполнителя.

### 1. Имена компонентов: правильные vs обманчивые

| Что нужно | ❌ Не используй | ✅ Используй |
|---|---|---|
| KPI-плитка (число) | `HsKpiPanel` | `HsTotalsPanelV2` |
| Таблица с строками | `HsTable`, `HsPivotTable` | `HsTableReactabularChart` |
| Текстовый блок | `HsTextEditorPanel` | `HsTextBlock` |

В старых версиях MCP (до фикса bug #1 в `builders.py:180`) неверный `component_type` **silently подменялся на `barChart`**, в отчёте появлялся пустой бар-чарт без warning. После фикса — возвращается ошибка с `available_types`.

### 2. Таблица: aggregation = `"max"`, не `"none"`

Несмотря на «логичный» `aggregation="none"` для таблиц, до фикса bug #3 frontend всё равно генерировал `SELECT MAX(...) FROM (...)` без `GROUP BY` — Postgres падал с `column must appear in GROUP BY`.

**Workaround на старом MCP:** не передавать `aggregation` вообще или явно `aggregation="max"`. Тогда `MAX(...) GROUP BY category` отрабатывает корректно.

**После фикса (`"Table" not in component_type` в builders.py)** — `aggregation="none"` работает как ожидается, но для безопасности skill всё равно передаёт `"max"` — это совместимо с обеими версиями MCP.

### 3. `HsTextBlock` через `add` — должен попадать в `_NO_DATASET_TYPES`

До фикса bug #2 вызов `component_crud(action="add", component_type="HsTextBlock", ...)` падал на валидации «Missing required parameters: value_field, dataset_id». После фикса — работает напрямую. **Совместимый рабочий путь** — всегда использовать специальное действие `component_crud(action="add_text_block", title=..., text_content=..., position={...})`.

### 4. `tabs_button_add` — параметр `title` не пробрасывается в JSON-schema

См. `_add_tabs_buttons.py` workaround. fastmcp не сериализует `Annotated[str | None, Field(...)] = None` — в MCP клиенте параметр становится «отсутствующим», и серверный `require()` падает. Прямой Python-вызов `component_operations(...)` минуя MCP-обёртку — **единственный** надёжный путь, пока bug #4 не исправлен (`title: str = ""` без `| None`).

### 5. `HsTextBlock` клампится в 88px без `modified: true` + `group: -1`

`add_text_block_component` сохраняет `position.h` в backend корректно (например h=14, h=26 во всех 6 breakpoints). Но **frontend ModusBI рендерит блок с фиксированной высотой 88px**, если в layout entry отсутствуют флаги `modified: true` и `group: -1`. Юзер при ручной правке через UI добавляет эти поля — после этого фронт начинает уважать h.

**Калибровка h** (1 grid row ≈ 22px):
- короткий lede / 1-строчный callout: `h=5` (110px)
- 2-3 предложения / список 3-4 пункта: `h=7-10`
- секция с h3 + таблица 3-5 строк: `h=18-22`
- секция с h3 + таблица 6-9 строк: `h=24-30`
- секция с code-block 10-20 строк: `h=20-30`

**Фикс применён** в `text_block_helper.py:108-114` (поля проставляются автоматически при каждом создании). Если используется старая версия helper — пройтись по layouts и установить `modified=true`, `group=-1` для всех HsTextBlock entries.

### Mini-checklist для skill-исполнителя

```python
# ✅ Корректные паттерны
component_crud(action="add",
    component_type="HsTotalsPanelV2",   # ← НЕ HsKpiPanel
    title="DQ Score", dataset_id=..., value_field="dq_score", ...)

component_crud(action="add",
    component_type="HsTableReactabularChart",  # ← НЕ HsTable
    aggregation="max",                  # ← НЕ "none" (см. bug #3)
    category_field="dimension",          # ← одно поле, не comma-separated
    value_field="score,weight,contribution",  # ← остальные поля как values
    ...)

component_crud(action="add_text_block",  # ← специальное действие, не "add"
    title="Lede", text_content="<h2>...</h2>",
    position={"x":0,"y":0,"w":60,"h":4})

# ❌ Грабли
component_crud(action="add",
    component_type="HsKpiPanel",        # → silently barChart
    ...)
component_crud(action="add",
    component_type="HsTableReactabularChart",
    aggregation="none",                 # → MAX без GROUP BY → 500
    category_field="dimension", ...)
```

---

## DAMA Reference

Подробнее про 5 dimensions и веса — см. [`dq-profiler/SKILL.md`](../dq-profiler/SKILL.md) §A.7.
