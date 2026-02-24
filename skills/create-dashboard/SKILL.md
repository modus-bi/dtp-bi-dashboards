---
name: create-dashboard
description: Создаёт полноценный BI дашборд с компонентами (KPI, графики, таблицы) и AI выводами. Используйте когда пользователь просит "создать дашборд", "построить отчёт", "визуализировать данные". Автоматически исследует источник данных, создаёт датасеты с правильным SQL, выбирает типы графиков, делает layout, генерирует Level 4 выводы.
license: MIT
metadata:
    skill-author: ModusBI Team
    version: "1.0"
    domain: business-intelligence
    mcp-servers: postgres, modusbi
---

# Create Dashboard: Создание полного BI дашборда

## Overview

Этот skill создаёт полноценный BI дашборд "под ключ":
- Исследует данные (если источник — PostgreSQL)
- Создаёт датасеты с оптимальным SQL
- Выбирает правильные типы графиков
- Делает автоматический layout
- Генерирует Level 4 выводы для каждого графика

**Результат:** Готовый дашборд с компонентами и бизнес-выводами за 5 минут.

**5-этапный workflow:**
1. Исследование данных (postgres-mcp) — понять что есть
2. Создание датасетов (modusbi-mcp) — SQL → датасет
3. Выбор компонентов (bi-visualization-expert) — правильные типы viz
4. Layout (auto-positioning) — красивое расположение
5. Генерация выводов (data-analyst-expert) — Level 4 narratives

---

## When to Use This Skill

Используй этот skill когда пользователь:

- ✅ Просит **"создать дашборд"** или **"построить отчёт"**
- ✅ Хочет **"визуализировать"** данные
- ✅ Спрашивает **"покажи в виде графика"**
- ✅ Нужен **комплексный дашборд** (не просто один график)
- ✅ Упоминает **KPI**, **метрики**, **показатели**
- ✅ Говорит про **период** (Q4, 2024, последний месяц)

**Триггерные фразы:**
- "Создай дашборд продаж за Q4"
- "Построй отчёт по регионам"
- "Визуализируй данные о ДТП"
- "Покажи KPI за месяц"
- "Операционный дашборд для команды"

**Не использовать если:**
- ❌ Нужен только SQL (используй `/generate-sql`)
- ❌ Нужен только анализ без viz (используй `/analyze-data`)
- ❌ Нужно объяснить существующий график (используй `/explain-chart`)

---

## Standard Workflow (5 этапов)

### Этап 1: Понять требования и источник данных

**Вопросы для clarification:**
```
1. Источник данных:
   - Таблица PostgreSQL? (имя таблицы/схемы)
   - Существующий датасет ModusBI? (ID или имя)

2. Что показать:
   - Какие метрики? (выручка, количество, маржа)
   - Какая группировка? (по регионам, месяцам, категориям)
   - Какой период? (2024, Q4, последние 12 месяцев)

3. Тип дашборда:
   - Executive (для руководства) — крупные KPI, минимум деталей
   - Operational (для менеджеров) — real-time, детализация
   - Analytical (для аналитиков) — много графиков, drill-down
```

**Если пользователь дал короткий промпт:**
```
User: "Создай дашборд продаж"

Agent: "Уточните:
- За какой период? (Q4 2024, весь 2024, последние 12 месяцев?)
- Какие метрики показать? (выручка, количество заказов, маржа?)
- Для кого дашборд? (руководство, менеджеры, аналитики?)

Или я могу создать стандартный дашборд с:
- KPI панель (выручка, заказы, средний чек)
- Динамика по месяцам (линейный график)
- Топ-10 регионов (столбчатая диаграмма)
- Таблица деталей"
```

---

### Этап 2: Исследование источника (postgres-mcp)

**Если источник — таблица PostgreSQL:**

```python
# 2.1 Получить схему
schema = postgres_mcp.query("""
    SELECT column_name, data_type
    FROM information_schema.columns
    WHERE table_schema = 'public'
      AND table_name = 'sales'
    ORDER BY ordinal_position
""")

# 2.2 Примеры данных (spot-check!)
sample = postgres_mcp.query("""
    SELECT * FROM sales
    LIMIT 10
""")

# 2.3 Базовая статистика
stats = postgres_mcp.query("""
    SELECT
        COUNT(*) as rows,
        COUNT(DISTINCT region) as regions,
        MIN(date) as from_date,
        MAX(date) as to_date,
        SUM(amount) as total_revenue
    FROM sales
""")

print(f"""
📊 ДАТАСЕТ: sales
• Строк: {rows:,}
• Регионов: {regions}
• Период: {from_date} — {to_date}
• Выручка: {total_revenue:,.0f} ₽

Поля: {', '.join(schema.column_name)}
""")
```

---

### Этап 3: Создание датасетов (modusbi-mcp)

**Используй правила из skill `/sql-bi-expert`:**

```python
# 3.1 SQL для основного датасета
sql = """
SELECT
    DATE_TRUNC('month', date) AS "Месяц",
    region AS "Регион",
    category AS "Категория",
    SUM(amount) AS "Выручка (₽)",
    ROUND(AVG(margin::numeric), 2) AS "Маржа (%)",
    COUNT(*) AS "Количество заказов"
FROM sales
WHERE date >= '{start_date}'
  AND date < '{end_date}'
  AND region IS NOT NULL
GROUP BY DATE_TRUNC('month', date), region, category
ORDER BY DATE_TRUNC('month', date), SUM(amount) DESC
"""

# 3.2 Создать датасет
dataset_id = modusbi_mcp.dataset_create_from_sql(
    sql_query=sql.format(start_date='2024-01-01', end_date='2025-01-01'),
    datasource_id=27,  # ID источника данных
    name="sales_2024_main",
    caption="Продажи 2024: основной датасет"
)

# 3.3 Настроить semantic layer (опционально, но рекомендуется)
modusbi_mcp.dataset_operations(
    action="update",
    dataset_id=dataset_id,
    field_aliases={
        "month": {
            "alias": "Месяц",
            "description": "Месяц продажи (группировка по DATE_TRUNC)"
        },
        "revenue": {
            "alias": "Выручка (₽)",
            "description": "Сумма выручки без НДС"
        }
    }
)
```

**Критичные правила SQL:**
- ✅ `ORDER BY` для сортировки (иначе порядок случайный!)
- ✅ `::INTEGER` для EXTRACT (чистые числа)
- ✅ `ROUND(..., 2)` для AVG и процентов
- ✅ `AS "Русские названия"` для легенд
- ✅ `WHERE ... IS NOT NULL` для фильтра пустых

---

### Этап 4: Создание компонентов (modusbi-mcp)

**Выбор компонентов** (из skill `/bi-visualization-expert`):

| Данные | Компонент | Когда |
|--------|-----------|-------|
| Ключевые метрики (1-5 чисел) | **HsTotalsPanelV2** | KPI для executive |
| Время + число | **HsLineAMchartsChart** | Тренды, динамика |
| Категория + число | **HsBarAMchartsChart** | Сравнение, топ-N |
| Доли (3-7 категорий) | **HsPieAMchartsChart** | Распределение |
| Много полей | **HsTableReactabularChart** | Детали, raw data |

**Workflow:**

```python
# 4.1 Создать отчёт
report = modusbi_mcp.report_operations(
    action="create",
    name="Sales Dashboard 2024",
    caption="Дашборд продаж за 2024",
    group_id=20  # ID группы отчётов
)
report_id = report['report_id']

# 4.2 KPI Panel (сверху)
modusbi_mcp.component_operations(
    action="add",
    report_id=report_id,
    component_type="HsTotalsPanelV2",
    title="Ключевые показатели",
    dataset_id=dataset_id,
    value_field="revenue,margin,orders",  # 3 метрики
    aggregation="sum,avg,count",          # Разные агрегации
    position={x: 0, y: 0, w: 24, h: 6}   # Full width, top
)

# 4.3 Line Chart (динамика) + AI вывод
modusbi_mcp.component_operations(
    action="add_chart_with_conclusion",  # График + текстовый блок!
    report_id=report_id,
    component_type="HsLineAMchartsChart",
    title="Динамика продаж по месяцам",
    dataset_id=dataset_id,
    category_field="month",      # ТЕХНИЧЕСКОЕ имя (не "Месяц"!)
    value_field="revenue",       # ТЕХНИЧЕСКОЕ имя
    aggregation="sum",
    text_content=generate_level4_conclusion(data),  # AI вывод
    h=16  # Общая высота для пары график+текст
)
# → График: {x: 0, y: 6, w: 12, h: 16}
# → Текст:  {x: 12, y: 6, w: 12, h: 16} (автоматически!)

# 4.4 Bar Chart (топ-10 регионов)
modusbi_mcp.component_operations(
    action="add",
    report_id=report_id,
    component_type="HsBarAMchartsChart",
    title="Топ-10 регионов",
    dataset_id=dataset_id,
    category_field="region",
    value_field="revenue",
    aggregation="sum",
    position={x: 0, y: 22, w: 12, h: 10}
)

# 4.5 Table (детализация)
modusbi_mcp.component_operations(
    action="add",
    report_id=report_id,
    component_type="HsTableReactabularChart",
    title="Детализация по менеджерам",
    dataset_id=dataset_id,
    value_field="manager,region,revenue,orders",  # Несколько полей
    value_fields_type="string",  # Для таблицы все как string
    aggregation="none",          # Raw data, без агрегации
    field_titles={               # Человекочитаемые заголовки
        "manager": "Менеджер",
        "region": "Регион",
        "revenue": "Выручка (₽)",
        "orders": "Заказов"
    },
    position={x: 12, y: 22, w: 12, h: 10}
)
```

**Layout pattern (F-pattern):**
```
┌────────────────────────────┐
│ KPI Panel (w=24, h=6)      │ y=0  ← Первый взгляд
├─────────────┬──────────────┤
│ Line Chart  │ AI Вывод     │ y=6  ← Второй взгляд
│ (w=12, h=16)│ (w=12, h=16) │
├─────────────┼──────────────┤
│ Bar Chart   │ Table        │ y=22 ← Третий взгляд
│ (w=12, h=10)│ (w=12, h=10) │
└─────────────┴──────────────┘
```

---

### Этап 5: Генерация выводов (data-analyst-expert)

**Для каждого значимого графика:**

```python
def generate_level4_conclusion(chart_data: dict) -> str:
    """
    Генерирует Level 4 вывод (prescriptive) для графика

    Опирается на:
    - /data-analyst-expert (структура вывода)
    - /statistical-analysis (проверка significance)
    """

    # Анализ данных
    analysis = analyze_chart_data(chart_data)

    # Формулировка вывода
    narrative = f"""
    <div style="background: linear-gradient(135deg, #e3f2fd, #bbdefb);
                padding: 20px; border-radius: 10px;
                border-left: 5px solid #1976d2;">

        <h2 style="color: #0d47a1;">📊 {analysis.title}</h2>

        <p><strong>Главная метрика:</strong> {analysis.main_metric}</p>

        <h3 style="color: #1565c0;">ДРАЙВЕРЫ РОСТА</h3>
        <ul>
            {"".join(f"<li><strong>{d.name}</strong> — {d.contribution}% ({d.explanation})</li>"
                     for d in analysis.drivers)}
        </ul>

        <h3 style="color: #c62828;">⚠️ ЗОНЫ РИСКА</h3>
        <ul>
            {"".join(f"<li><strong>{r.problem}</strong> — {r.scale} ({r.consequence})</li>"
                     for r in analysis.risks)}
        </ul>

        <h3 style="color: #2e7d32;">💡 РЕКОМЕНДАЦИИ</h3>
        <ol>
            {"".join(f"<li><strong>[{a.priority}]</strong> {a.action}<br/>"
                     f"Дедлайн: {a.deadline}, Impact: {a.impact}</li>"
                     for a in analysis.actions)}
        </ol>

        <p><strong>ПРОГНОЗ:</strong> {analysis.forecast}</p>
    </div>
    """

    return narrative
```

---

## Common Patterns

### Pattern 1: Simple Dashboard (базовый)

**User input:**
```
/create-dashboard "продажи за Q4 2024"
```

**Output:**
```python
# Создаём стандартный дашборд:
# 1. KPI панель (сверху)
# 2. Line chart динамики (слева)
# 3. Bar chart топ-10 (справа)
# 4. Table детализация (внизу)

report_id = create_report("Sales Q4 2024", group_id=20)

# KPI
add_component(
    type="HsTotalsPanelV2",
    value_field="revenue,orders,avg_check",
    position={x: 0, y: 0, w: 24, h: 6}
)

# Line + Conclusion
add_chart_with_conclusion(
    type="HsLineAMchartsChart",
    title="Динамика по месяцам",
    category_field="month",
    value_field="revenue",
    h=16  # График + текст
)

# Bar Chart
add_component(
    type="HsBarAMchartsChart",
    title="Топ-10 регионов",
    category_field="region",
    value_field="revenue",
    position={x: 0, y: 22, w: 12, h: 10}
)

# Table
add_component(
    type="HsTableReactabularChart",
    value_field="date,client,amount,status",
    position={x: 12, y: 22, w: 12, h: 10}
)
```

**Результат:** Дашборд с 4 компонентами за 3-5 минут

---

### Pattern 2: Executive Dashboard (для руководства)

**User input:**
```
/create-dashboard "executive dashboard для директора: KPI, тренд, топ-10"
```

**Особенности:**
```python
# Executive требует:
# - Крупные цифры (KPI 48-72px)
# - Минимум деталей
# - Фокус на ключевые метрики
# - Сравнение план/факт

# KPI с крупными цифрами
add_component(
    type="HsTotalsPanelV2",
    title="",  # Без заголовка (больше места для цифр)
    value_field="revenue,profit,roi",
    position={x: 0, y: 0, w: 24, h: 8}  # Выше обычного (h=8 vs h=6)
)

# Один главный график (не перегружаем)
add_chart_with_conclusion(
    type="HsLineAMchartsChart",
    title="Динамика выручки",
    category_field="month",
    value_field="revenue,plan",  # Факт + План
    h=18
)

# Топ-5 (не топ-10, меньше деталей)
add_component(
    type="HsBarAMchartsChart",
    title="Топ-5 направлений",
    limit=5,  # Только 5!
    position={x: 0, y: 26, w: 24, h: 8}
)
```

**Стиль:**
```python
# Применить executive тему
modusbi_mcp.style_operations(
    action="update",
    report_id=report_id,
    css="""
    .kpi-value { font-size: 48px !important; }
    .kpi-label { font-size: 18px !important; }
    .header { background: #0d47a1 !important; color: white !important; }
    """
)
```

---

### Pattern 3: Analytical Dashboard (для аналитиков)

**User input:**
```
/create-dashboard "детальный анализ продаж: все метрики, drill-down, корреляции"
```

**Особенности:**
```python
# Analytical требует:
# - Много графиков (6-9)
# - Детализация
# - Drill-down возможности
# - Корреляции
# - Raw data (таблицы)

# Компоненты:
# 1. KPI (compact)
# 2. Line chart (main metric)
# 3-4. Bar charts (breakdown по 2 dimensions)
# 5. Scatter plot (correlation)
# 6. Heatmap (если применимо)
# 7-8. Tables (детали)

# Layout: 3 колонки (w=8 каждая)
```

---

## Best Practices

### 1. Выбор компонентов по данным

**Decision tree:**
```
Что показываем?
│
├─ 1-5 ключевых чисел
│  └─ HsTotalsPanelV2 (KPI panel)
│
├─ Динамика по времени
│  └─ HsLineAMchartsChart (линейный график)
│
├─ Сравнение категорий
│  ├─ <15 категорий → HsBarAMchartsChart (столбчатая)
│  └─ >15 категорий → HsTableReactabularChart (таблица)
│
├─ Пропорции/доли
│  ├─ 3-7 категорий → HsPieAMchartsChart (круговая)
│  └─ >7 категорий → HsBarAMchartsChart (столбчатая)
│
└─ Детали/много полей
   └─ HsTableReactabularChart (таблица)
```

### 2. Layout Hierarchy (F-pattern)

**Размещай по важности:**

```
Важность:  1 > 2 > 3 > 4

┌─────────────────────┐
│  1. KPI (самое      │ ← Первый взгляд
│     важное)         │
├──────────┬──────────┤
│ 2. Main  │ 3. Sec-  │ ← Второй взгляд
│    chart │    ondary│
├──────────┴──────────┤
│ 4. Details (table)  │ ← Третий взгляд
└─────────────────────┘

Position:
1. {x: 0, y: 0}   (top-left)
2. {x: 0, y: 6}   (left)
3. {x: 12, y: 6}  (right)
4. {x: 0, y: 18}  (bottom)
```

### 3. Auto-positioning Helper

**Используй helper для автоматического y:**

```python
def get_next_y(report_id: int) -> int:
    """Найти свободное место внизу дашборда"""
    components = modusbi_mcp.component_operations(
        action="list",
        report_id=report_id
    )

    if not components:
        return 0

    return max(c['y'] + c['h'] for c in components)

# Использование:
position = {x: 0, y: get_next_y(report_id), w: 12, h: 8}
```

### 4. Всегда добавляй выводы к главным графикам

**Используй `add_chart_with_conclusion`:**

```python
# ✅ ХОРОШО: График + вывод
add_chart_with_conclusion(
    type="HsLineAMchartsChart",
    title="Динамика продаж",
    text_content=level4_narrative  # AI вывод справа
)

# 🟡 OK: Только график (для простых KPI)
add_component(
    type="HsBarAMchartsChart",
    title="Топ-10 товаров"
    # Без вывода (очевиден)
)
```

**Когда добавлять выводы:**
- ✅ Главные графики (тренды, важные сравнения)
- ✅ Сложные анализы (корреляции, прогнозы)
- ✅ Неочевидные паттерны
- ❌ Простые KPI (самоочевидны)
- ❌ Таблицы (данные говорят сами)

### 5. Ограничивай количество компонентов

**Правило 7±2:**

Человек может держать в фокусе **7±2 элемента**.

```
Executive dashboard:  4-5 компонентов
Operational:          5-7 компонентов
Analytical:           7-9 компонентов (максимум!)

Больше 9 → разбивай на несколько дашбордов
```

---

## Examples

### Example 1: Sales Dashboard (полный)

**Input:**
```
/create-dashboard "дашборд продаж за 2024: KPI, динамика, топ-10 регионов, детали"
```

**Workflow:**

```python
# 1. Исследование (если нужно)
schema = postgres_mcp.describe_table("sales")

# 2. Датасет
dataset_id = modusbi_mcp.dataset_create_from_sql(
    sql_query="""
        SELECT
            DATE_TRUNC('month', date) AS month,
            region,
            SUM(amount) AS revenue,
            ROUND(AVG(margin::numeric), 2) AS margin,
            COUNT(*) AS orders
        FROM sales
        WHERE date >= '2024-01-01' AND date < '2025-01-01'
          AND region IS NOT NULL
        GROUP BY DATE_TRUNC('month', date), region
        ORDER BY DATE_TRUNC('month', date)
    """,
    datasource_id=27,
    name="sales_2024"
)

# 3. Отчёт
report_id = modusbi_mcp.report_operations(
    action="create",
    name="Sales Dashboard 2024",
    group_id=20
)['report_id']

# 4. Компоненты (5 штук)

# 4.1 KPI
modusbi_mcp.component_operations(
    action="add",
    component_type="HsTotalsPanelV2",
    report_id=report_id,
    title="KPI",
    dataset_id=dataset_id,
    value_field="revenue,margin,orders",
    aggregation="sum,avg,count",
    position={x: 0, y: 0, w: 24, h: 6}
)

# 4.2 Line Chart + Conclusion
modusbi_mcp.component_operations(
    action="add_chart_with_conclusion",
    report_id=report_id,
    component_type="HsLineAMchartsChart",
    title="Динамика продаж по месяцам",
    dataset_id=dataset_id,
    category_field="month",
    value_field="revenue",
    aggregation="sum",
    text_content="""
        <div style="background: linear-gradient(135deg, #e3f2fd, #bbdefb);
                    padding: 20px; border-radius: 10px;
                    border-left: 5px solid #1976d2;">
            <h2 style="color: #0d47a1;">📊 ДИНАМИКА ПРОДАЖ 2024</h2>

            <p><strong>Выручка:</strong> 156.7M ₽ (+12% г/г)</p>

            <h3 style="color: #1565c0;">ДРАЙВЕРЫ РОСТА</h3>
            <ul>
                <li><strong>Электроника</strong> — +34% (вклад 67%)</li>
                <li><strong>Москва</strong> — +28% (вклад 23%)</li>
            </ul>

            <h3 style="color: #2e7d32;">💡 РЕКОМЕНДАЦИИ</h3>
            <ol>
                <li><strong>[КРИТИЧНО]</strong> Расширить ассортимент электроники → +3-4M ₽</li>
            </ol>
        </div>
    """,
    h=16
)

# 4.3-5 Bar Chart + Table
# [остальные компоненты...]

print(f"✅ Дашборд создан! ID: {report_id}")
print(f"   Компонентов: 5")
print(f"   Ссылка: https://bi.company.com/report/{report_id}")
```

**Time:** ~5 минут

---

### Example 2: Quick Dashboard (быстрый)

**Input:**
```
/create-dashboard "топ-10 клиентов по выручке"
```

**Workflow (упрощённый):**
```python
# Создаём минимальный дашборд: 1 график

# 1. SQL
dataset_id = create_dataset("""
    SELECT
        client AS "Клиент",
        SUM(amount) AS "Выручка (₽)"
    FROM sales
    WHERE date >= '2024-01-01'
    GROUP BY client
    ORDER BY SUM(amount) DESC
    LIMIT 10
""")

# 2. Отчёт + график
report_id = create_report("Top 10 Clients")
add_component(
    type="HsBarAMchartsChart",
    title="Топ-10 клиентов",
    dataset_id=dataset_id,
    category_field="client",
    value_field="revenue"
)

print(f"✅ График готов! {report_id}")
```

**Time:** ~2 минуты

---

### Pattern 3: Multi-Dataset Dashboard (сложный)

**User input:**
```
/create-dashboard "комплексный анализ: продажи + маркетинг + retention"
```

**Workflow:**
```python
# 3 датасета для разных метрик

# Dataset 1: Продажи
sales_ds = create_dataset("SELECT ... FROM sales ...")

# Dataset 2: Маркетинг
marketing_ds = create_dataset("SELECT ... FROM marketing ...")

# Dataset 3: Retention
retention_ds = create_dataset("SELECT ... FROM cohorts ...")

# Дашборд с компонентами из разных датасетов
report_id = create_report("Complex Analysis")

add_component(..., dataset_id=sales_ds)
add_component(..., dataset_id=marketing_ds)
add_component(..., dataset_id=retention_ds)

# Relationships (если нужны связи между датасетами)
modusbi_mcp.relationship_operations(
    action="create",
    from_dataset=sales_ds,
    to_dataset=marketing_ds,
    on_field="date"
)
```

---

## Dashboard Templates (готовые шаблоны)

### Template 1: Sales Dashboard

**Компоненты:**
1. KPI: Revenue, Orders, Avg Check
2. Line: Revenue by month (trend)
3. Bar: Top 10 regions
4. Pie: Category split
5. Table: Recent orders

**SQL:**
```sql
SELECT
    DATE_TRUNC('month', date) AS month,
    region,
    category,
    SUM(amount) AS revenue,
    COUNT(*) AS orders,
    ROUND(AVG(amount::numeric), 2) AS avg_check
FROM sales
WHERE date >= '2024-01-01'
GROUP BY month, region, category
ORDER BY month
```

---

### Template 2: Marketing Dashboard

**Компоненты:**
1. KPI: CAC, LTV, ROI, Conversion
2. Funnel: Lead → Qual → Offer → Deal
3. Bar: ROI by channel
4. Line: CAC trend
5. Table: Campaign details

---

### Template 3: HR Dashboard

**Компоненты:**
1. KPI: Headcount, Turnover, Avg Tenure
2. Line: Turnover by month
3. Heatmap: Turnover by dept × manager
4. Bar: Reasons for leaving
5. Table: At-risk employees

---

## Troubleshooting

### Issue: Компонент не показывает данные

**Symptom:** Пустой график

**Решение:**
```python
# Проверка:
# 1. Датасет существует?
dataset = modusbi_mcp.dataset_operations(action="get", dataset_id=...)
print(f"Dataset: {dataset}")

# 2. Поля правильные?
fields = modusbi_mcp.dataset_operations(action="fields_list", dataset_id=...)
print(f"Available fields: {[f.name for f in fields]}")

# 3. Используешь ТЕХНИЧЕСКИЕ имена (не aliases)?
# ❌ category_field="Регион"  # Alias (неправильно!)
# ✅ category_field="region"   # Техническое имя

# 4. SQL вернул данные?
data = modusbi_mcp.dataset_query(dataset_id=...)
print(f"Rows: {len(data.rows)}")
```

---

### Issue: Компоненты наслаиваются

**Symptom:** Графики перекрывают друг друга

**Решение:**
```python
# Проверь позиции:
components = modusbi_mcp.component_operations(
    action="list",
    report_id=report_id
)

for c in components:
    print(f"{c.title}: x={c.x}, y={c.y}, w={c.w}, h={c.h}")
    # Проверь: x + w ≤ 24 (не выходит за границу)
    # Проверь: нет перекрытий

# Используй get_next_y() для автоматического y
```

---

### Issue: SQL слишком медленный

**Symptom:** Timeout при создании датасета

**Решение:**
```sql
-- 1. Добавь WHERE для периода
WHERE date >= '2024-01-01'  -- Не весь архив!

-- 2. Добавь LIMIT
LIMIT 1000  -- Для больших таблиц

-- 3. Используй indexed поля в WHERE
WHERE indexed_field = value  -- Быстрее

-- 4. Избегай:
-- ❌ SELECT *
-- ❌ Коррелированные подзапросы
-- ❌ CROSS JOIN

-- 5. Проверь EXPLAIN:
EXPLAIN ANALYZE SELECT ...
-- Execution time должно быть <10 сек
```

---

## Reference Documentation

### Связанные Skills (детальные руководства)

- **/sql-bi-expert** — 30+ SQL паттернов, правила ORDER BY, типы
- **/bi-visualization-expert** — Матрица выбора viz, layout, цвета
- **/data-analyst-expert** — Level 4 выводы, структура рекомендаций
- **/prompt-engineering-bi** — Как формулировать требования к дашборду

### ModusBI API

- Типы компонентов: `component_operations(action="types_list")`
- CSS классы: `style_operations(action="classes_list")`
- Примеры стилей: `style_operations(action="examples_list")`

---

## Quick Reference

### Стандартные размеры

```javascript
// KPI Panel
{x: 0, y: 0, w: 24, h: 6}

// Charts (графики)
{x: 0, y: 6, w: 12, h: 10}  // Половина ширины

// Chart + Conclusion (add_chart_with_conclusion)
h: 16  // Общая высота (график и текст)

// Table (таблица)
{x: 0, y: 22, w: 24, h: 12}  // Full width
```

### Типовые датасеты

```sql
-- Time series (динамика)
SELECT DATE_TRUNC('month', date), SUM(value)
GROUP BY DATE_TRUNC('month', date)
ORDER BY DATE_TRUNC('month', date)  -- ОБЯЗАТЕЛЬНО!

-- Top-N (топ)
SELECT category, SUM(value)
GROUP BY category
ORDER BY SUM(value) DESC
LIMIT 10  -- ОБЯЗАТЕЛЬНО!

-- Comparison (сравнение)
SELECT category,
       SUM(CASE WHEN period = 'A' THEN value END) as period_a,
       SUM(CASE WHEN period = 'B' THEN value END) as period_b
GROUP BY category
```

---

## Important Notes

### Используй технические имена полей!

```python
# ❌ НЕПРАВИЛЬНО:
category_field="Регион"  # Alias (русское название)

# ✅ ПРАВИЛЬНО:
category_field="region"  # Техническое имя из SQL
```

**Почему:** ModusBI API требует технические имена. Alias только для отображения.

### Semantic Layer — после создания датасета

```python
# 1. Создаём датасет с техническими именами
dataset_id = dataset_create_from_sql(
    sql="SELECT region, amount FROM sales"
)

# 2. Потом настраиваем semantic layer
dataset_operations(
    action="update",
    dataset_id=dataset_id,
    field_aliases={
        "region": {"alias": "Регион", "description": "Географический регион РФ"},
        "amount": {"alias": "Выручка (₽)", "description": "Сумма без НДС"}
    }
)

# 3. При создании компонента используем ТЕХНИЧЕСКИЕ имена
component_operations(..., category_field="region")  # НЕ "Регион"!
```

