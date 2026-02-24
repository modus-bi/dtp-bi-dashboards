---
name: bi-visualization-expert
description: "BI Visualization Expert: выбор правильного типа графика, настройка компонентов и layout дашборда. Используйте когда нужно выбрать тип графика для данных, настроить компоненты дашборда, спроектировать layout отчёта, применить best practices визуализации."
license: MIT
metadata:
    skill-author: ModusBI Team
    version: "1.0"
    domain: business-intelligence
    mcp-servers: modusbi
---

# BI Visualization Expert: Правильный выбор и настройка графиков

> Skill для выбора типа визуализации и правильной настройки компонентов дашборда

## Назначение

Используй этот skill когда нужно:
- Выбрать тип графика для данных
- Настроить компоненты дашборда
- Спроектировать layout отчёта
- Применить best practices визуализации
- Создать effective dashboard

## Матрица выбора типа графика

### Вопрос: "Какие данные визуализируем?"

| Тип данных | Задача | Рекомендуемый график |
|------------|--------|---------------------|
| **Одна категория + число** | Сравнение | Bar Chart (столбчатая) |
| **Время + число** | Тренд | Line Chart (линейный) |
| **Категории + доли** | Пропорции | Pie Chart (круговая) |
| **Много полей** | Детали | Table (таблица) |
| **Несколько чисел** | KPI | Totals Panel (панель) |
| **Категория + время** | Динамика по группам | Stacked Area/Line |
| **Две числовые переменные** | Корреляция | Scatter Plot |
| **География** | Региональное распределение | Map / Choropleth |

---

## Типы графиков: когда что использовать

### 1. Bar Chart (HsBarAMchartsChart) — столбчатая диаграмма

**Используй когда:**
- Сравниваешь категории (регионы, товары, менеджеры)
- Топ-N (топ-10 клиентов, лучшие продукты)
- Категорий 3-15 (не слишком мало, не слишком много)

**НЕ используй когда:**
- Динамика по времени (используй Line Chart)
- Слишком много категорий (>20 — используй Table)
- Пропорции/доли (используй Pie Chart)

**Пример:**
```javascript
component_operations(
  action: "add",
  component_type: "HsBarAMchartsChart",
  title: "Топ-10 регионов по продажам",
  dataset_id: 342,
  category_field: "region",       // X-axis (категории)
  value_field: "revenue",         // Y-axis (значения)
  aggregation: "sum",             // Агрегация
  position: {x: 0, y: 0, w: 12, h: 8}
)
```

**Best practices:**
- Сортируй по убыванию (самый большой слева)
- Используй горизонтальную если названия длинные
- Limit 10-15 категорий максимум
- Добавь data labels для точных значений

---

### 2. Line Chart (HsLineAMchartsChart) — линейный график

**Используй когда:**
- Динамика по времени (продажи по месяцам)
- Тренды (рост/падение)
- Несколько линий для сравнения (план/факт, регионы)

**НЕ используй когда:**
- Категории без временной последовательности
- Пропорции (используй Pie)

**Пример:**
```javascript
component_operations(
  action: "add",
  component_type: "HsLineAMchartsChart",
  title: "Динамика продаж по месяцам",
  dataset_id: 342,
  category_field: "month",        // X-axis (ОБЯЗАТЕЛЬНО ORDER BY!)
  value_field: "revenue",         // Y-axis
  aggregation: "sum"
)
```

**Best practices:**
- ВСЕГДА ORDER BY date в SQL (иначе линия будет ломаной!)
- Используй несколько линий для сравнения (факт vs план)
- Добавь markers для важных точек
- Smooth line для трендов (если много шума)

---

### 3. Pie Chart (HsPieAMchartsChart) — круговая диаграмма

**Используй когда:**
- Показать доли/пропорции (% от целого)
- Категорий 3-7 (не более!)
- Нужно показать "из чего состоит"

**НЕ используй когда:**
- Много категорий (>7 — нечитаемо)
- Тренды (используй Line)
- Сравнение абсолютных значений (используй Bar)

**Пример:**
```javascript
component_operations(
  action: "add",
  component_type: "HsPieAMchartsChart",
  title: "Распределение продаж по категориям",
  dataset_id: 342,
  category_field: "category",
  value_field: "revenue_pct",     // Проценты!
  aggregation: "sum"
)
```

**Best practices:**
- Limit 5-7 категорий (группируй остальное в "Прочие")
- Сортируй от большего к меньшему (clockwise)
- Используй контрастные цвета
- Добавь проценты на сегменты

**SQL для Pie Chart:**
```sql
SELECT
  category AS "Категория",
  ROUND(SUM(amount)::numeric / SUM(SUM(amount)) OVER () * 100, 1) AS "Доля (%)"
FROM sales
GROUP BY category
ORDER BY SUM(amount) DESC
LIMIT 7
```

---

### 4. Table (HsTableReactabularChart) — таблица

**Используй когда:**
- Много полей (5+)
- Нужны детали (не агрегированные данные)
- Точные числа важны
- Нужна возможность поиска/фильтрации

**НЕ используй когда:**
- Нужно показать тренд (используй график)
- Сравнение категорий (используй Bar)

**Пример:**
```javascript
component_operations(
  action: "add",
  component_type: "HsTableReactabularChart",
  title: "Детализация заказов",
  dataset_id: 342,
  value_field: "date,client,product,amount,status",  // Несколько полей!
  value_fields_type: "string",    // Все как string (для таблицы)
  aggregation: "none",            // Нет агрегации (raw data)
  field_titles: {                 // Человекочитаемые заголовки
    "date": "Дата",
    "client": "Клиент",
    "product": "Товар",
    "amount": "Сумма (₽)",
    "status": "Статус"
  }
)
```

**Best practices:**
- Limit 100-200 строк (пагинация)
- Сортировка по дате (newest first)
- Форматирование чисел (запятые для тысяч)
- Подсветка важных значений (conditional formatting)

---

### 5. Totals Panel (HsTotalsPanelV2) — панель KPI

**Используй когда:**
- Нужно показать ключевые метрики (KPI)
- Один или несколько больших чисел
- Сравнение с baseline (план/факт, прошлый период)

**Пример:**
```javascript
component_operations(
  action: "add",
  component_type: "HsTotalsPanelV2",
  title: "Ключевые показатели",
  dataset_id: 342,
  value_field: "revenue,margin,orders",  // Несколько метрик
  aggregation: "sum,avg,count"           // Разные агрегации
)
```

**Best practices:**
- 3-5 метрик максимум (не перегружай)
- Крупный шрифт (48-72px)
- Цвет для статуса (зелёный=хорошо, красный=плохо)
- Добавь change indicator (up 12% vs прошлый месяц)

---

### 6. Area Chart (HsAreaAMchartsChart) — диаграмма с областями

**Используй когда:**
- Динамика по времени + показать объём
- Stacked area: несколько категорий показать вместе
- Акцент на накопительный эффект

**Пример:**
```javascript
component_operations(
  action: "add",
  component_type: "HsAreaAMchartsChart",
  title: "Структура продаж по категориям",
  dataset_id: 342,
  category_field: "month",
  value_field: "electronics,clothing,food",  // Несколько областей
  aggregation: "sum"
)
```

---

## Layout: Grid System (24 колонки)

### Правила позиционирования

```
Grid: 24 колонки (как Bootstrap)

Стандартные ширины:
- Full width: w=24 (вся ширина)
- Half: w=12 (половина)
- Third: w=8 (треть)
- Quarter: w=6 (четверть)

Высота (h):
- KPI panel: h=4-6
- График: h=8-12
- Таблица: h=10-16
```

### Типовые layouts

**Layout 1: KPI + 2 графика + таблица**
```
+----------------------------+
| KPI Panel (w=24, h=6)      | y=0
+-------------+--------------+
| Chart 1     | Chart 2      | y=6
| (w=12, h=8) | (w=12, h=8)  |
+-------------+--------------+
| Table (w=24, h=12)         | y=14
+----------------------------+
```

**Код:**
```javascript
// KPI
{x: 0, y: 0, w: 24, h: 6}

// Chart 1 (слева)
{x: 0, y: 6, w: 12, h: 8}

// Chart 2 (справа)
{x: 12, y: 6, w: 12, h: 8}

// Table (внизу)
{x: 0, y: 14, w: 24, h: 12}
```

---

**Layout 2: Chart + Conclusion (автопозиционирование)**

```javascript
// Используй add_chart_with_conclusion для автоматического layout!
component_operations(
  action: "add_chart_with_conclusion",
  component_type: "HsBarAMchartsChart",
  title: "График",
  text_content: "<div>Вывод...</div>",
  h: 16  // Общая высота
)

// Результат:
// График: {x: 0, y: auto, w: 12, h: 16}
// Текст:  {x: 12, y: auto, w: 12, h: 16}
```

**Автоматическое y:**
```python
def get_next_y_position(report_id: int) -> int:
    """Найти свободное место внизу дашборда"""
    components = get_components(report_id)
    if not components:
        return 0
    return max(c.y + c.h for c in components)
```

---

## Цветовые схемы

### Стандартные палитры

**Корпоративная (синяя):**
```css
.header { background: #1976d2; }
.accent { color: #42a5f5; }
.chart-line { stroke: #1976d2; }
```

**Для презентации (контрастная):**
```css
.header { background: #1565c0; color: white; }
.positive { color: #4caf50; }  /* Зелёный для роста */
.negative { color: #f44336; }  /* Красный для падения */
```

**Для печати (ч/б):**
```css
.chart-line { stroke: #000000; stroke-width: 2px; }
.background { background: white; }
.text { color: #333333; }
```

---

### Семантические цвета (используй правильно!)

| Цвет | Значение | Использование |
|------|----------|---------------|
| Зелёный (#4caf50) | Позитив, рост | Revenue up, targets met |
| Красный (#f44336) | Негатив, падение | Revenue down, alerts |
| Синий (#1976d2) | Нейтрально, информация | Headers, main color |
| Жёлтый (#ffc107) | Внимание, warning | Approaching limits |
| Оранжевый (#ff9800) | Средний приоритет | Moderate risks |
| Серый (#757575) | Неактивно, второстепенно | Disabled, secondary |

**Пример использования:**
```html
<div style="color: #4caf50;">+12% рост</div>
<div style="color: #f44336;">-8% падение</div>
<div style="color: #ff9800;">Близко к лимиту</div>
```

---

## Dashboard Layout Principles

### Принцип 1: F-Pattern (F-паттерн чтения)

Пользователи читают дашборд **F-образно:**
1. Сверху слева -> направо (горизонтально)
2. Вниз по левому краю (вертикально)
3. Снова вправо (второй горизонтальный проход)

**Оптимальное размещение:**
```
+-----------------------------+
| 1. KPI (САМОЕ ВАЖНОЕ)       | <- Первый взгляд
+--------------+--------------+
| 2. Главный   | 4. Допол-    |
|    график    |    нительный | <- Второй взгляд
+--------------+--------------+
| 5. Детали (таблица)         | <- Третий взгляд
+-----------------------------+

Приоритет:
1 > 2 > 4 > 5
```

---

### Принцип 2: Visual Hierarchy (визуальная иерархия)

**Размер = важность:**

```
KPI (самое важное):     w=24, h=6  (полная ширина)
Главные графики:        w=12, h=10 (половина ширины)
Второстепенные графики: w=8, h=8   (треть ширины)
Детали (таблица):       w=24, h=12 (полная ширина, внизу)
```

**Группировка:**
```
Связанные компоненты рядом:
- "Продажи" и "Маржа" -> side by side
- "План" и "Факт" -> side by side
- "Топ товары" и "Детали по товарам" -> один под другим
```

---

### Принцип 3: Whitespace (воздух между элементами)

**НЕ делай плотно:**
```
Плохо:
+------+------+------+
|Chart1|Chart2|Chart3|  <- Всё слиплось
+------+------+------+
```

**Делай с отступами:**
```
Хорошо:
+------+  +------+  +------+
|Chart1|  |Chart2|  |Chart3|  <- Есть воздух
+------+  +------+  +------+
```

**Реализация:**
```javascript
// Используй w=11 вместо w=12 для отступа
{x: 0, y: 0, w: 11, h: 8}   // Chart 1
{x: 12, y: 0, w: 11, h: 8}  // Chart 2 (есть gap 1 колонка)
```

---

## Настройка компонентов: Best Practices

### Bar Chart Settings

```javascript
{
  component_type: "HsBarAMchartsChart",
  title: "Краткий заголовок (3-5 слов)",
  dataset_id: 342,

  // Поля (используй ТЕХНИЧЕСКИЕ названия из датасета!)
  category_field: "region",      // НЕ "Регион", а "region"
  value_field: "revenue",        // НЕ "Выручка", а "revenue"

  // Агрегация
  aggregation: "sum",  // sum, avg, count, min, max

  // Позиция
  position: {
    x: 0,   // Левый край (0-23)
    y: 0,   // Верх
    w: 12,  // Ширина (половина = 12)
    h: 8    // Высота
  }
}
```

**Важно:**
- `category_field` и `value_field` — **ТЕХНИЧЕСКИЕ** имена (из SQL)
- "Регион" (alias) задаётся через `field_aliases` в датасете
- LLM должен использовать оригинальные имена полей!

---

### Line Chart Settings

```javascript
{
  component_type: "HsLineAMchartsChart",
  title: "Динамика по месяцам",
  dataset_id: 342,

  category_field: "month",       // X-axis (ДОЛЖЕН быть ORDER BY!)
  value_field: "revenue",        // Y-axis

  // Для нескольких линий
  value_field: "revenue,orders", // Через запятую

  aggregation: "sum"
}
```

**Чек-лист:**
- [ ] SQL имеет ORDER BY по category_field
- [ ] category_field — дата/время для трендов
- [ ] Не более 3-5 линий (иначе нечитаемо)

---

### Table Settings

```javascript
{
  component_type: "HsTableReactabularChart",
  title: "Детализация",
  dataset_id: 342,

  // Несколько полей через запятую
  value_field: "date,client,product,amount,status",

  // Тип полей (важно!)
  value_fields_type: "string",  // Все как строки для таблицы

  // Без агрегации (показать raw data)
  aggregation: "none",

  // Человекочитаемые заголовки
  field_titles: {
    "date": "Дата",
    "client": "Клиент",
    "product": "Товар",
    "amount": "Сумма (₽)",
    "status": "Статус"
  },

  // Limit в SQL (не более 200 строк)
  limit: 100
}
```

---

### Totals Panel Settings

```javascript
{
  component_type: "HsTotalsPanelV2",
  title: "KPI",
  dataset_id: 342,

  // Несколько метрик
  value_field: "revenue,orders,clients",

  // Разные агрегации для каждой метрики
  aggregation: "sum,count,count_distinct"
}
```

---

## Создание текстовых блоков (выводы)

### HTML структура для выводов

**Шаблон:**
```html
<div style="
  background: linear-gradient(135deg, #e3f2fd, #bbdefb);
  padding: 20px;
  border-radius: 10px;
  border-left: 5px solid #1976d2;
  font-family: 'Segoe UI', Arial, sans-serif;
">

<h2 style="color: #0d47a1; margin-top: 0;">
  ЗАГОЛОВОК ВЫВОДА
</h2>

<p style="font-size: 16px; line-height: 1.6;">
  <strong>Главная метрика:</strong> 156.7M руб (+12% г/г)
</p>

<h3 style="color: #1565c0;">ДРАЙВЕРЫ РОСТА</h3>
<ul style="line-height: 1.8;">
  <li><strong>Электроника</strong> — +34% (вклад 67%)</li>
  <li><strong>Москва</strong> — +28% (вклад 23%)</li>
</ul>

<h3 style="color: #c62828;">ЗОНЫ РИСКА</h3>
<ul style="line-height: 1.8;">
  <li><strong>Одежда</strong> — -5% (потеря 2.3M руб)</li>
</ul>

<h3 style="color: #2e7d32;">РЕКОМЕНДАЦИИ</h3>
<ol style="line-height: 1.8;">
  <li><strong>[КРИТИЧНО]</strong> Найти замену Газпром -> +5.2M руб/мес</li>
  <li><strong>[ВЫСОКИЙ]</strong> Расширить Электронику -> +3-4M руб</li>
</ol>

</div>
```

---

### Цветовые схемы для выводов

**Позитивный (рост, success):**
```html
<div style="background: linear-gradient(135deg, #e8f5e9, #c8e6c9);
            border-left: 5px solid #4caf50;">
  <h2 style="color: #2e7d32;">ГИПОТЕЗА ПОДТВЕРЖДЕНА</h2>
  ...
</div>
```

**Негативный (риск, alert):**
```html
<div style="background: linear-gradient(135deg, #ffebee, #ffcdd2);
            border-left: 5px solid #f44336;">
  <h2 style="color: #c62828;">ПРОБЛЕМА ОБНАРУЖЕНА</h2>
  ...
</div>
```

**Нейтральный (информация):**
```html
<div style="background: linear-gradient(135deg, #e3f2fd, #bbdefb);
            border-left: 5px solid #1976d2;">
  <h2 style="color: #0d47a1;">АНАЛИЗ</h2>
  ...
</div>
```

---

## Выбор визуализации: Decision Tree

```
Что хочешь показать?
|
+- Сравнить категории
|  +- Категорий <15 -> Bar Chart
|  +- Категорий >15 -> Table
|
+- Показать тренд
|  +- Одна линия -> Line Chart
|  +- Несколько линий -> Line Chart (multiple)
|  +- Накопительный -> Area Chart (stacked)
|
+- Показать пропорции
|  +- Категорий 3-7 -> Pie Chart
|  +- Категорий >7 -> Bar Chart (horizontal)
|
+- Показать детали
|  +- Поля <5 -> Table (compact)
|  +- Поля >5 -> Table (full width)
|
+- Показать KPI
|  +- Метрик 1-3 -> Totals Panel (large)
|  +- Метрик 4-6 -> Totals Panel (grid)
|
+- Показать корреляцию
   +- Scatter Plot (если доступен)
   +- Line Chart (две линии)
```

---

## Чек-лист создания компонента

### Перед созданием графика проверь:

**Данные:**
- [ ] Датасет создан и содержит нужные поля
- [ ] SQL имеет ORDER BY (если важен порядок)
- [ ] SQL имеет LIMIT (для топ-N)
- [ ] Поля имеют правильные типы (INTEGER, numeric)
- [ ] NULL значения обработаны

**Настройки:**
- [ ] component_type правильный для задачи
- [ ] title краткий и понятный (3-5 слов)
- [ ] category_field и value_field — ТЕХНИЧЕСКИЕ имена
- [ ] aggregation соответствует задаче
- [ ] position не перекрывает другие компоненты

**Layout:**
- [ ] x, y, w, h в пределах grid (0-23 для x, 0+ для y)
- [ ] w + x <= 24 (не выходит за границу)
- [ ] h адекватен для типа (таблица выше, KPI ниже)

---

## Примеры готовых дашбордов

### Dashboard 1: Executive Dashboard (для руководства)

```
Компоненты:
1. KPI Panel (сверху): Revenue, Margin, Orders
   {x: 0, y: 0, w: 24, h: 6}

2. Line Chart (динамика): Revenue by month
   {x: 0, y: 6, w: 16, h: 10}

3. Text Block (выводы):
   {x: 16, y: 6, w: 8, h: 10}

4. Bar Chart (топ): Top 10 regions
   {x: 0, y: 16, w: 12, h: 10}

5. Pie Chart (распределение): Category split
   {x: 12, y: 16, w: 12, h: 10}

Стиль:
- Крупные цифры (48px для KPI)
- Минимум деталей (руководству не нужен noise)
- Акцент на ключевые метрики
```

---

### Dashboard 2: Operational Dashboard (для менеджеров)

```
Компоненты:
1. KPI Panel: Daily/Weekly/Monthly targets
   {x: 0, y: 0, w: 24, h: 4}

2. Line Chart: Sales trend (30 days)
   {x: 0, y: 4, w: 12, h: 8}

3. Bar Chart: Top managers
   {x: 12, y: 4, w: 12, h: 8}

4. Table: Recent orders (last 50)
   {x: 0, y: 12, w: 24, h: 14}

Стиль:
- Обновление real-time (или частое)
- Подсветка отклонений (красный/зелёный)
- Drill-down возможность
```

---

### Dashboard 3: Analytical Dashboard (для аналитиков)

```
Компоненты:
1. Line Chart: Main metric over time
   {x: 0, y: 0, w: 16, h: 10}

2. Text Block: Analysis & insights
   {x: 16, y: 0, w: 8, h: 10}

3. Bar Chart: Breakdown by dimension 1
   {x: 0, y: 10, w: 8, h: 10}

4. Bar Chart: Breakdown by dimension 2
   {x: 8, y: 10, w: 8, h: 10}

5. Scatter Plot: Correlation
   {x: 16, y: 10, w: 8, h: 10}

6. Table: Detailed data
   {x: 0, y: 20, w: 24, h: 12}

Стиль:
- Много деталей (аналитикам нужны подробности)
- Возможность экспорта (CSV/Excel)
- Интерактивные фильтры
```

---

## Антипаттерны визуализации

### Неправильный тип графика

**Проблема: Pie chart с 20 категориями**
```
Нечитаемо! Слишком много сегментов.

Решение:
- Топ-7 категорий + "Прочие"
- Или используй Bar Chart (horizontal)
```

**Проблема: Line chart для категорий**
```
"Продажи по регионам" через Line Chart — линия между регионами нелогична!

Решение: Bar Chart
```

---

### Слишком много на одном дашборде

**Плохо:**
```
15 графиков на одном дашборде
-> Информационный перегруз
-> Пользователь не знает куда смотреть
```

**Хорошо:**
```
5-7 компонентов максимум:
- 1 KPI panel
- 2-3 главных графика
- 1-2 второстепенных
- 1 таблица деталей
```

**Правило 7+-2:**
Человек может удержать в фокусе **7+-2** элемента.
Больше 9 компонентов -> разбивай на несколько дашбордов.

---

### Неправильные масштабы осей

**Проблема: Y-axis не начинается с 0**
```
Bar Chart с Y от 90 до 100:
-> Визуально выглядит как огромная разница
-> На самом деле разница 10%

Решение: Y-axis от 0 (или явно укажи если нет)
```

**Проблема: Разные масштабы для сравнения**
```
График 1: Revenue (0-100M)
График 2: Revenue (50-60M)
-> Визуально несравнимо!

Решение: Одинаковые масштабы для сравнимых графиков
```

---

## Adaptive Design: дашборд для разных устройств

### Desktop (основной)

```
Grid: 24 колонки
Ширина: 1920px

Компоненты:
- KPI: w=24 (full width)
- Charts: w=12 (2 в ряд)
- Tables: w=24 (full width)
```

### Tablet (если нужна адаптация)

```
Grid: 12 колонок (половина)

Компоненты:
- KPI: w=12 (full)
- Charts: w=12 (1 в ряд, друг под другом)
```

### Mobile (обычно не нужно для BI)

```
BI дашборды редко смотрят на mobile.
Если нужно -> создавай отдельный "мобильный дашборд":
- Только KPI
- Простые графики
- Минимум деталей
```

---

## Специализированные визуализации

### Heatmap (тепловая карта)

**Используй для:**
- Correlation matrix (корреляции между метриками)
- Cohort analysis (retention по когортам)
- Time patterns (активность по часам x дням недели)

**SQL для heatmap:**
```sql
-- Активность по часам и дням недели
SELECT
  EXTRACT(DOW FROM timestamp)::INTEGER AS day_of_week,
  EXTRACT(HOUR FROM timestamp)::INTEGER AS hour,
  COUNT(*) AS activity
FROM events
WHERE timestamp >= NOW() - INTERVAL '30 days'
GROUP BY day_of_week, hour
ORDER BY day_of_week, hour
```

---

### Funnel Chart (воронка)

**Используй для:**
- Conversion funnel (лид -> квал -> оффер -> сделка)
- User journey (регистрация -> активация -> покупка)

**SQL для funnel:**
```sql
SELECT
  stage AS "Этап",
  stage_order,
  COUNT(*) AS "Количество",
  LAG(COUNT(*)) OVER (ORDER BY stage_order) AS prev_stage,
  ROUND(COUNT(*)::numeric / LAG(COUNT(*)) OVER (ORDER BY stage_order) * 100, 1) AS "Конверсия (%)"
FROM funnel
GROUP BY stage, stage_order
ORDER BY stage_order
```

---

### Gauge Chart (спидометр)

**Используй для:**
- Progress к цели (выполнение плана)
- Single metric с min/max/target

**Пример:**
```javascript
{
  type: "GaugeChart",
  value: 87,        // Текущее значение
  min: 0,
  max: 100,
  target: 100,      // Целевое значение
  ranges: [
    {from: 0, to: 70, color: "red"},      // Плохо
    {from: 70, to: 90, color: "yellow"},  // Средне
    {from: 90, to: 100, color: "green"}   // Хорошо
  ]
}
```

---

## Визуализация по типам анализа

### Trend Analysis -> Line Chart

```
Задача: Показать динамику
График: Line Chart
X-axis: Время (месяц/день)
Y-axis: Метрика
Best practice: Добавить trendline, MA (скользящее среднее)
```

### Comparison Analysis -> Bar Chart

```
Задача: Сравнить категории
График: Bar Chart (horizontal если названия длинные)
X-axis: Категория
Y-axis: Значение
Best practice: Сортировка DESC, топ-10
```

### Distribution Analysis -> Histogram / Box Plot

```
Задача: Показать распределение
График: Histogram (если доступен)
Альтернатива: Table с percentiles (p25, p50, p75, p95)
```

### Correlation Analysis -> Scatter Plot

```
Задача: Показать связь двух метрик
График: Scatter Plot
X-axis: Метрика 1
Y-axis: Метрика 2
Best practice: Добавить regression line (линия тренда)
```

### Composition Analysis -> Pie Chart / Stacked Bar

```
Задача: Показать из чего состоит
График: Pie Chart (если категорий <7)
Альтернатива: Stacked Bar Chart (если категорий >7)
```

---

## Продвинутые техники

### Multiple Axes (несколько осей)

**Когда использовать:**
- Две метрики с разными масштабами (Revenue: млн, Orders: тысячи)

**Пример:**
```javascript
{
  type: "HsLineAMchartsChart",
  value_field: "revenue,orders",
  aggregation: "sum,count",
  axes: {
    revenue: {position: "left", label: "Выручка (руб)"},
    orders: {position: "right", label: "Заказов (шт)"}
  }
}
```

---

### Conditional Formatting (условное форматирование)

**Для таблиц:**
```javascript
{
  type: "HsTableReactabularChart",
  conditional_formatting: [
    {
      field: "revenue",
      condition: "> 1000000",
      style: {background: "#e8f5e9", color: "#2e7d32"}  // Зелёный для >1M
    },
    {
      field: "margin",
      condition: "< 10",
      style: {background: "#ffebee", color: "#c62828"}  // Красный для <10%
    }
  ]
}
```

---

### Drill-down (углубление)

**Концепция:**
```
Дашборд 1 (высокоуровневый):
"Продажи по регионам" (Bar Chart)
-> Клик на "Москва"
-> Переход на Дашборд 2

Дашборд 2 (детальный):
"Продажи Москва: по магазинам, товарам, менеджерам"
```

**Реализация через фильтры:**
```javascript
// Дашборд 1
{
  component_type: "HsBarAMchartsChart",
  category_field: "region",
  on_click: {
    action: "filter",
    target_dashboard: 598,
    filter_field: "region"
  }
}
```

---

## Стандартные размеры компонентов

### Рекомендации по высоте (h)

```javascript
// KPI Panel
h: 4-6   (компактно, только цифры)

// Charts (графики)
h: 8-12  (стандартная высота)

// Charts с легендой
h: 10-14 (нужно место для legend)

// Tables (небольшие)
h: 10-12 (10-15 строк)

// Tables (полные)
h: 14-20 (20-50 строк)

// Text blocks (выводы)
h: 8-16  (зависит от объёма текста)
```

---

### Рекомендации по ширине (w)

```javascript
// Full width (заголовки, таблицы)
w: 24

// Half (два графика рядом)
w: 12

// Third (три графика рядом)
w: 8

// Quarter (четыре маленьких KPI)
w: 6

// График + Текст (add_chart_with_conclusion)
График: w: 12, Текст: w: 12
```

---

## Готовые шаблоны компонентов

### Template 1: Chart with Conclusion

```javascript
component_operations(
  action: "add_chart_with_conclusion",
  report_id: 598,
  component_type: "HsBarAMchartsChart",
  title: "Топ-10 регионов по продажам",
  dataset_id: 342,
  category_field: "region",
  value_field: "revenue",
  aggregation: "sum",
  text_content: `
    <div style="background: linear-gradient(135deg, #e3f2fd, #bbdefb);
                padding: 20px; border-radius: 10px; border-left: 5px solid #1976d2;">
      <h2 style="color: #0d47a1;">ВЫВОДЫ</h2>
      <p>Топ-3 региона дают 67% выручки...</p>
      <h3 style="color: #2e7d32;">РЕКОМЕНДАЦИИ</h3>
      <ol>
        <li>Усилить маркетинг в регионах 4-10</li>
        <li>Диверсифицировать риски топ-3</li>
      </ol>
    </div>
  `,
  h: 16  // Общая высота
)

// Автоматически создаст:
// График: {x: 0, y: auto, w: 12, h: 16}
// Текст:  {x: 12, y: auto, w: 12, h: 16}
```

---

### Template 2: KPI Panel

```javascript
component_operations(
  action: "add",
  component_type: "HsTotalsPanelV2",
  title: "Ключевые показатели",
  dataset_id: 342,
  value_field: "revenue,margin,orders,clients",
  aggregation: "sum,avg,count,count_distinct",
  position: {x: 0, y: 0, w: 24, h: 6}
)
```

---

### Template 3: Time Series with Comparison

```javascript
component_operations(
  action: "add",
  component_type: "HsLineAMchartsChart",
  title: "Динамика продаж: 2024 vs 2023",
  dataset_id: 342,
  category_field: "month",
  value_field: "revenue_2024,revenue_2023",  // Две линии
  aggregation: "sum",
  position: {x: 0, y: 6, w: 16, h: 10}
)
```

---

## Когда использовать add_chart_with_conclusion

### Критерии

**Используй когда:**
- График требует объяснения (что это значит)
- Нужны рекомендации на основе данных
- Сложный анализ (пользователь может не понять)
- Prescriptive analytics (что делать)

**НЕ используй когда:**
- График очевиден (простой KPI panel)
- Нет выводов (просто показать данные)
- Места мало (w < 24 для пары)

### Примеры

**Хороший use case:**
```
График: "Корреляция времени доставки и NPS"
Вывод: "Каждый день задержки -> -0.7 NPS.
        Рекомендация: SLA 1-2 дня -> +1.5 NPS -> +12% retention"

-> add_chart_with_conclusion
```

**Плохой use case:**
```
График: "Продажи сегодня"
Вывод: "Продажи сегодня 2.3M"

-> Просто показать число, не нужен вывод
-> Используй Totals Panel
```

---

## Color Palettes для разных тем

### Корпоративная (синяя)

```css
/* Primary */
--primary: #1976d2;
--primary-light: #42a5f5;
--primary-dark: #0d47a1;

/* Accent */
--accent: #ff9800;

/* Background */
--bg: #fafafa;
--bg-card: #ffffff;

/* Text */
--text-primary: #212121;
--text-secondary: #757575;

/* Semantic */
--success: #4caf50;
--warning: #ffc107;
--error: #f44336;
```

---

### Презентация (контрастная)

```css
/* High contrast для проектора */
--primary: #0d47a1;      /* Тёмно-синий */
--accent: #ff5722;       /* Яркий оранжевый */
--bg: #ffffff;           /* Белый фон */
--text: #000000;         /* Чёрный текст */

/* Charts */
--chart-1: #1976d2;
--chart-2: #f44336;
--chart-3: #4caf50;
--chart-4: #ff9800;
```

---

### Печать (чёрно-белая)

```css
/* Grayscale для печати */
--primary: #212121;
--secondary: #757575;
--bg: #ffffff;

/* Charts (оттенки серого) */
--chart-1: #212121;
--chart-2: #616161;
--chart-3: #9e9e9e;

/* Borders для разделения */
border: 1px solid #000000;
```

---

## Best Practices Summary

### DO (делай)

- **Выбирай правильный тип графика** для типа данных
- **Сортируй данные** (ORDER BY в SQL!)
- **Ограничивай количество** (LIMIT, топ-N)
- **Используй цвет семантически** (зелёный=рост, красный=падение)
- **Добавляй выводы** к графикам (text blocks с рекомендациями)
- **Группируй связанное** (sales chart + sales table рядом)
- **Оставляй whitespace** (не всё впритык)
- **Приоритезируй layout** (важное вверху слева)

### DON'T (не делай)

- **Не перегружай** дашборд (максимум 7-9 компонентов)
- **Не используй 3D** (тяжело читать, искажает данные)
- **Не используй слишком много цветов** (максимум 5-7)
- **Не делай pie chart** с >7 категориями
- **Не забывай про ORDER BY** в SQL!
- **Не используй SELECT *** без необходимости
- **Не перекрывай компоненты** (проверяй x, y, w, h)

---

## Финальный чек-лист дашборда

### Перед публикацией проверь:

**Данные:**
- [ ] Все компоненты показывают данные (нет ошибок)
- [ ] Графики в правильном порядке (ORDER BY)
- [ ] Числа правильно округлены (ROUND)
- [ ] Названия понятные (AS "Русские названия")

**Визуализация:**
- [ ] Типы графиков подходят для данных
- [ ] Количество компонентов адекватно (5-9)
- [ ] Цвета семантически правильные
- [ ] Layout логичный (важное вверху)

**UX:**
- [ ] Заголовки понятные (без жаргона)
- [ ] Единицы измерения указаны (руб, шт, %)
- [ ] Есть выводы/рекомендации (text blocks)
- [ ] Whitespace между компонентами

**Performance:**
- [ ] Дашборд загружается <3 сек
- [ ] Нет огромных таблиц (>200 строк)
- [ ] Нет тяжёлых SQL (timeout <10 сек)

---

*Используй этот skill для создания effective BI dashboards*
