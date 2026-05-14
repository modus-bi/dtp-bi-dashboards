---
name: dq-rules-catalog
description: Reference-каталог стандартных DQ-правил по типам данных (ИНН/КПП/ОГРН/email/phone/даты/координаты). Используется `/dq-profiler` для авто-генерации правил по семантике колонок. Содержит regex, threshold-ы, severity-defaults для российской и общей специфики.
license: MIT
metadata:
    skill-author: ModusBI Team
    version: "1.0"
    domain: data-quality
    mcp-servers: postgres
user-invocable: true
---

# DQ Rules Catalog: каталог стандартных правил

## Overview

Reference-скилл со списком готовых DQ-правил. Используется `/dq-profiler` для **семантической авто-генерации** правил по именам колонок. Например, колонка `inn` → автоматически применяется regex `^\d{10}$|^\d{12}$` + checksum-валидация.

**Зачем:** LLM без подсказок генерирует regex для ИНН правильно лишь в ~60% случаев (забывает про 12-значный формат ИП, не знает алгоритм checksum). Этот каталог — инструкция, которая закрывает гэп.

---

## When to Use

- Внутри `/dq-profiler` Этап B (выработка правил).
- Когда автор пишет SKILL.md и нужны regex-эталоны.
- Когда нужно объяснить «откуда взялось правило DQM-XXX».

---

## Каталог по семантике колонок

### A. Идентификаторы РФ

#### A1. ИНН (Идентификационный номер налогоплательщика)

| | |
|---|---|
| Имя колонки (matchers) | `inn`, `INN`, `taxpayer_id`, `taxpayer_number` |
| Regex | `^\d{10}$\|^\d{12}$` (10 для юр.лиц, 12 для ИП/ФЛ) |
| Checksum | Алгоритм ФНС — взвешенная сумма цифр |
| Severity | CRITICAL |
| Типичная грязь | Ведущие нули потеряны (CSV); смешение ИНН компании и ИП |

```sql
-- Базовая валидация формата
WHERE inn IS NULL OR inn !~ '^\d{10}$|^\d{12}$'

-- Полная (с checksum для 10-значного):
-- 1. Веса: 2,4,10,3,5,9,4,6,8 (для первых 9 цифр)
-- 2. Сумма % 11, если >= 10 → 0
-- 3. Сравнить с 10-й цифрой
WITH parsed AS (
    SELECT inn,
           CAST(SUBSTRING(inn, 1, 1) AS int) * 2 +
           CAST(SUBSTRING(inn, 2, 1) AS int) * 4 +
           CAST(SUBSTRING(inn, 3, 1) AS int) * 10 +
           CAST(SUBSTRING(inn, 4, 1) AS int) * 3 +
           CAST(SUBSTRING(inn, 5, 1) AS int) * 5 +
           CAST(SUBSTRING(inn, 6, 1) AS int) * 9 +
           CAST(SUBSTRING(inn, 7, 1) AS int) * 4 +
           CAST(SUBSTRING(inn, 8, 1) AS int) * 6 +
           CAST(SUBSTRING(inn, 9, 1) AS int) * 8 AS checksum_sum,
           CAST(SUBSTRING(inn, 10, 1) AS int) AS checksum_digit
    FROM <table> WHERE LENGTH(inn) = 10
)
SELECT COUNT(*) FILTER (
    WHERE (checksum_sum % 11) % 10 != checksum_digit
) AS checksum_violations
FROM parsed;
```

#### A2. КПП (Код причины постановки на учёт)

| | |
|---|---|
| Имя | `kpp`, `KPP` |
| Regex | `^\d{4}([\dA-Z]{2}\|[А-Я]{2})\d{3}$` (9 знаков; 5–6 могут быть буквы — латиница для НКО, кириллица для иностр.) |
| Альтернатива (либеральная) | `^\d{4}.{2}\d{3}$` |
| Severity | HIGH |

```sql
WHERE kpp !~ '^\d{4}([\dA-Z]{2}|[А-Я]{2})\d{3}$'
```

#### A3. ОГРН/ОГРНИП

| | |
|---|---|
| Имя | `ogrn`, `OGRN`, `ogrnip` |
| Regex | `^\d{13}$` (юр.лиц) или `^\d{15}$` (ИП) |
| Checksum | Остаток от деления на 11/13 |
| Severity | HIGH |

```sql
WHERE ogrn !~ '^\d{13}$|^\d{15}$'
-- Checksum 13: ABS(SUBSTRING(ogrn, 1, 12)::bigint % 11) % 10 = LAST_DIGIT
```

#### A4. БИК (банковский идентификационный код)

| | |
|---|---|
| Имя | `bik`, `BIK` |
| Regex | `^0\d{8}$` (9 знаков, начинается с 0 — Положение Банка России 762-П) |
| Severity | HIGH |

#### A5. Расчётный счёт

| | |
|---|---|
| Имя | `account_number`, `bank_account` |
| Regex | `^\d{20}$` |
| Checksum | Алгоритм ЦБ (по таблице весов) |
| Severity | CRITICAL для платёжных операций |

#### A6. СНИЛС

| | |
|---|---|
| Имя | `snils`, `SNILS` |
| Regex (формат) | `^\d{11}$\|^\d{3}-\d{3}-\d{3} \d{2}$\|^\d{3}-\d{3}-\d{3}-\d{2}$` |
| Checksum | Взвешенная сумма с весами 9..1 для первых 9 цифр, mod 101; если sum == 100 или 101 → 00 |
| Severity | MEDIUM |

```sql
-- Формат + checksum
WITH parsed AS (
    SELECT REGEXP_REPLACE(snils, '[^\d]', '', 'g') AS digits
    FROM <table> WHERE snils IS NOT NULL
),
checksum AS (
    SELECT digits,
        CAST(SUBSTRING(digits, 1, 1) AS int) * 9 +
        CAST(SUBSTRING(digits, 2, 1) AS int) * 8 +
        CAST(SUBSTRING(digits, 3, 1) AS int) * 7 +
        CAST(SUBSTRING(digits, 4, 1) AS int) * 6 +
        CAST(SUBSTRING(digits, 5, 1) AS int) * 5 +
        CAST(SUBSTRING(digits, 6, 1) AS int) * 4 +
        CAST(SUBSTRING(digits, 7, 1) AS int) * 3 +
        CAST(SUBSTRING(digits, 8, 1) AS int) * 2 +
        CAST(SUBSTRING(digits, 9, 1) AS int) * 1 AS s,
        CAST(SUBSTRING(digits, 10, 2) AS int) AS check_digits
    FROM parsed WHERE LENGTH(digits) = 11
)
SELECT COUNT(*) FILTER (
    WHERE NOT (
        (s < 100 AND s = check_digits)
        OR (s IN (100, 101) AND check_digits = 0)
        OR (s > 101 AND (s % 101) = check_digits)
        OR (s > 101 AND (s % 101) IN (100, 101) AND check_digits = 0)
    )
) AS checksum_violations
FROM checksum;
```

---

### B. Контактные данные

#### B1. Email

| | |
|---|---|
| Имя | `email`, `e_mail` |
| Regex | `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$` |
| Severity | MEDIUM |
| Типичная грязь | Trailing/leading whitespace; `noreply@example.com` placeholder |

#### B2. Телефон РФ

| | |
|---|---|
| Имя | `phone`, `phone_number`, `tel` |
| Regex (E.164) | `^\+7\d{10}$` |
| Regex (либеральный) | `^[\+\(\)\s\d\-]+$` (для импорта) |
| Severity | MEDIUM |
| Типичная грязь | Разные форматы — `8(495)...`, `+7 495 ...`, `(495) ...` |

```sql
-- Нормализация перед проверкой
WHERE REGEXP_REPLACE(phone, '[^\d+]', '', 'g') !~ '^\+7\d{10}$|^8\d{10}$'
```

#### B3. URL/website

| | |
|---|---|
| Имя | `url`, `website`, `homepage` |
| Regex | `^https?://[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}.*$` |
| Severity | LOW |

---

### C. Адреса

#### C1. ФИАС UUID

| | |
|---|---|
| Имя | `n_fias`, `fias_id`, `fias_uuid` |
| Regex | `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` (case-insensitive) |
| Severity | MEDIUM |

#### C2. КЛАДР

| | |
|---|---|
| Имя | `kladr`, `kladr_id` |
| Regex | `^\d{17}$` |
| Severity | LOW |

#### C3. Кадастровый номер

| | |
|---|---|
| Имя | `kad_n`, `kadastr_n`, `cadastre_number` |
| Regex | `^\d{1,2}:\d{1,2}:\d{6,7}:\d+$` (первые два блока — 1-2 цифры, не строго 2) |
| Severity | HIGH |

#### C4. Почтовый индекс РФ

| | |
|---|---|
| Имя | `postal_code`, `zip`, `index` |
| Regex | `^\d{6}$` |
| Severity | LOW |

---

### D. Геокоординаты

#### D1. Latitude

| | |
|---|---|
| Имя | `lat`, `latitude` |
| Range | `-90.0 ≤ lat ≤ 90.0` |
| RU-bbox | `41.0 ≤ lat ≤ 82.0` |
| Severity | HIGH (если "0,0" — sentinel-проблема) |

#### D2. Longitude

| | |
|---|---|
| Имя | `lon`, `lng`, `longitude` |
| Range | `-180.0 ≤ lon ≤ 180.0` |
| RU-bbox | `19.0 ≤ lon ≤ 180.0` (с Калининградом) |

```sql
WHERE lat NOT BETWEEN 41.0 AND 82.0
   OR lon NOT BETWEEN 19.0 AND 180.0
   OR (lat = 0 AND lon = 0)  -- sentinel
```

---

### E. Даты и времена

#### E1. Дата ISO

| | |
|---|---|
| Имя | `*_date`, `created_at`, `updated_at` |
| Тип | `date`, `timestamp`, `timestamptz` |
| Range | `1900-01-01 ≤ x ≤ NOW() + INTERVAL '1 year'` |
| Severity | MEDIUM |

```sql
WHERE x < '1900-01-01' OR x > NOW() + INTERVAL '1 year'
```

#### E2. Дата в свободной форме (ОКН/исторические)

| | |
|---|---|
| Имя | `create_date`, `creation_date` (для культурных объектов) |
| Парсинг | YYYY / YYYY-YYYY / "конец XIX в." / "XVIII век" |
| Severity | LOW (если поле культурное) |

```sql
WHERE create_date IS NOT NULL
  AND create_date !~ '^\d{4}$'
  AND create_date !~ '^\d{4}-\d{4}$'
  AND create_date !~ '(век|в\.|XV|XVI|XVII|XVIII|XIX|XX|XXI)'
```

#### E3. Дата DD.MM.YYYY (русский формат)

| | |
|---|---|
| Имя | `dreg`, `ddoc`, `*_date` (если значения strings) |
| Regex | `^\d{2}\.\d{2}\.\d{4}$` |
| Парсинг | `to_date(x, 'DD.MM.YYYY')` |
| Severity | MEDIUM |

---

### F. Деньги, проценты, числа

#### F1. Сумма

| | |
|---|---|
| Имя | `amount`, `total`, `price`, `cost` |
| Range | `0 ≤ x ≤ 1e12` (триллион — sane limit для одной транзакции) |
| Severity | HIGH (выбросы дороже не-нулей) |

```sql
WHERE amount < 0 OR amount > 1e12
```

#### F2. Процент

| | |
|---|---|
| Имя | `*_pct`, `*_percent`, `rate` |
| Range | `0 ≤ x ≤ 100` |
| Severity | MEDIUM |

#### F3. Год

| | |
|---|---|
| Имя | `year`, `year_built`, `*_year` |
| Range | `1700 ≤ x ≤ EXTRACT(YEAR FROM NOW()) + 1` |
| Severity | MEDIUM |

```sql
WHERE year < 1700 OR year > EXTRACT(YEAR FROM NOW()) + 1
```

---

### G. Системные

#### G1. UUID

| | |
|---|---|
| Имя | `*_id` (если type = uuid), `guid` |
| Regex | `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` |
| Severity | LOW |

#### G2. Boolean (text fields)

| | |
|---|---|
| Имя | `is_*`, `has_*` (если text) |
| Допустимые | `true/false`, `yes/no`, `да/нет`, `1/0` |
| Severity | LOW |

#### G3. Enum-поля

| | |
|---|---|
| Имя | `status`, `type`, `category` |
| Проверка | значение IN (allowed list) |
| Severity | HIGH (нарушение enum часто — баг ETL) |

```sql
WHERE status NOT IN ('active', 'archived', 'deleted', 'pending')
```

---

## Метаправила (cross-field)

### M1. Уникальность ключа

| | |
|---|---|
| Применимо | PK-кандидаты (`id`, любое поле с uniqueness 100%) |
| Правило | `GROUP BY <field> HAVING COUNT(*) > 1` |
| Severity | CRITICAL |

### M2. FK-целостность

| | |
|---|---|
| Применимо | `<entity>_id` ↔ `<entity>.id` |
| Правило | `LEFT JOIN ... WHERE parent.id IS NULL` |
| Severity | CRITICAL |

### M3. Кросс-поле сумма

| | |
|---|---|
| Применимо | `total = qty × price` |
| Правило | `WHERE ABS(total - qty * price) > 0.01` |
| Severity | HIGH |

### M4. Дата-инвариант

| | |
|---|---|
| Применимо | `end_date >= start_date` |
| Правило | `WHERE end_date < start_date` |
| Severity | CRITICAL |

### M5. Округ ↔ район

| | |
|---|---|
| Применимо | `district` ↔ `adm_area` (Москва) |
| Правило | `district` принадлежит ровно одному `adm_area` |
| Severity | MEDIUM |

```sql
WITH district_areas AS (
    SELECT district, COUNT(DISTINCT adm_area) AS distinct_areas
    FROM <table> GROUP BY district
)
SELECT COUNT(*) FROM district_areas WHERE distinct_areas > 1;
```

---

## Общие правила (применимы к любой колонке)

| Тип | Описание | Severity |
|---|---|---|
| **NOT NULL** | Все обязательные поля заполнены | зависит от поля |
| **Whitespace** | `field LIKE ' %' OR field LIKE '% '` (trailing/leading) | LOW |
| **Empty string** | `field = ''` (вместо NULL) | LOW |
| **Sentinel value** | `field IN ('-', 'N/A', 'unknown', 'NULL', 'null', '0', 'TBD')` | MEDIUM |
| **ОКПО** | `^\d{8}$\|^\d{10}$` (юр.лица 8, ИП 10) | LOW |
| **ОКВЭД** | `^\d{2}\.\d{1,2}(\.\d{1,2})?$` (например `47.30`, `47.30.1`) | LOW |
| **ОКТМО** | `^\d{8}$\|^\d{11}$` | LOW |
| **ОКАТО** | `^\d{2,11}$` | LOW |
| **VIN** | `^[A-HJ-NPR-Z0-9]{17}$` (без I, O, Q) | MEDIUM |
| **Госномер РФ** | `^[АВЕКМНОРСТУХ]\d{3}[АВЕКМНОРСТУХ]{2}\d{2,3}$` (только разрешённые кириллические) | LOW |
| **IBAN** | `^[A-Z]{2}\d{2}[A-Z0-9]{11,30}$` | LOW |
| **Encoding artifacts** | `field LIKE '%Ð%'` (broken UTF-8) | HIGH |
| **HTML in text** | `field ~ '<[^>]+>'` (нежелательный HTML) | MEDIUM |

---

## Статистические правила

| Тип | Описание | Применение |
|---|---|---|
| **3σ outlier** | `WHERE x < AVG - 3*STDDEV OR x > AVG + 3*STDDEV` | Числовые столбцы |
| **IQR outlier** | `WHERE x < Q1 - 1.5*IQR OR x > Q3 + 1.5*IQR` | Альтернатива 3σ для скошенных распределений |
| **Энтропия Шеннона** | `H > 5 бит` для категориальных = подозрение на грязь | Категории |
| **Levenshtein near-duplicate** | `levenshtein(LOWER(a), LOWER(b)) <= 3` для строк > 5 символов | Имена, названия |
| **Schema drift** | сравнение `information_schema.columns` снапшотов | Структура |

---

## Output Format

При запросе «дай regex для ИНН» / «правила для контрагентов» — возвращает:

```markdown
# DQM-правила: контрагенты (рекомендуемые)

## A. ИНН (DQM-001..DQM-003)
- Регекс: `^\d{10}$|^\d{12}$` — CRITICAL
- Checksum (для 10-зн): SQL → 5%
- Уникальность: GROUP BY HAVING > 1 — CRITICAL

## B. КПП (DQM-004)
- ...

## Сводная таблица
| ID | Поле | Тип | Severity | SQL |
|---|---|---|---|---|
| DQM-001 | inn | NOT_NULL | CRITICAL | `WHERE inn IS NULL` |
| ...
```

---

## Best Practices

1. **`severity` зависит от бизнеса** — этот каталог даёт defaults, но в банке `phone` может быть CRITICAL (для 2FA), а в каталоге товаров — LOW.
2. **Checksum дороже regex** — для ИНН/ОГРН/Bank account проверяй сначала regex (~99% мусора отсеивает), потом checksum (~остаток).
3. **Используй `~` (regex) с осторожностью на больших таблицах** — добавь индекс по `LENGTH(field)` если нет.
4. **Sentinel-значения** — особенно `0`, `'0'`, `null` (как строка), `'N/A'`. Их часто ошибочно считают валидными.
5. **Российская специфика** — большинство regex чувствительны к ведущим нулям; CSV/Excel их теряет → читай как `text`, не `int`.

---

## Связанные

- `/dq-profiler` — использует этот каталог для авто-генерации правил.
- `/create-dq-dashboard` — визуализирует результаты применения.
- `/generate-sql` — для создания custom-правил вне каталога.
