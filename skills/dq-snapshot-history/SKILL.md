---
name: dq-snapshot-history
description: Архивирует результаты DQ-аудита в `dq_history` для построения trend во времени. Используется как cron-задача или вручную после `/dq-profiler`. Принимает `dq_snapshots` и копирует строки с текущим `run_at` в архивную таблицу с retention. Универсален.
license: MIT
metadata:
    skill-author: ModusBI Team
    version: "1.0"
    domain: data-quality
    mcp-servers: postgres
user-invocable: true
---

# DQ Snapshot History: историзация DQ-метрик для Trend

## Overview

Создаёт и заполняет таблицу `dq_history` снапшотами текущего состояния `dq_snapshots`. Без этого скилла раздел `dq-trend` в `/create-dq-dashboard` остаётся пустым (требуется `COUNT(DISTINCT run_at) >= 2`).

**Зависит от:** `/dq-profiler` — для расчёта метрик в `dq_snapshots`.
**Используется:** `/create-dq-dashboard` (читает `dq_history`).

---

## When to Use

- После каждого прогона `/dq-profiler` для сохранения снапшота.
- В cron-задаче для регулярной историзации.
- Перед первым запуском `/create-dq-dashboard` — иначе раздел Trend не создаётся.

**Не использовать если:**
- В `dq_snapshots` нет `run_at` колонки.
- Не нужен trend (одноразовый аудит).

---

## Контракт

### Входы

| Параметр | Тип | Обязателен | Описание |
|---|---|---|---|
| `source_table` | text | ✓ | Таблица текущих результатов (например, `mos.dq_snapshots`) |
| `history_table` | text | — | Архив (default: `<source>_history`, например `mos.dq_snapshots_history`) |
| `retention_days` | int | — | Сколько дней хранить (default 365) |

### Выход

Запись в `history_table` всех строк из `source_table` с текущим `run_at`. Если такой `run_at` уже есть в history — пропуск (idempotent).

---

## Контракт `history_table`

Создаётся автоматически если не существует. Схема:

```sql
CREATE TABLE IF NOT EXISTS <history_table> (
    snapshot_id   bigserial PRIMARY KEY,
    archived_at   timestamptz DEFAULT now(),
    run_at        timestamptz NOT NULL,
    rule_id       text NOT NULL,
    rule_type     text NOT NULL,
    table_name    text NOT NULL,
    field_name    text,
    total_rows    bigint NOT NULL,
    violations    bigint NOT NULL,
    violation_pct numeric(5,2) GENERATED ALWAYS AS
        (CASE WHEN total_rows > 0 THEN 100.0 * violations / total_rows ELSE 0 END) STORED,
    severity      text,

    UNIQUE (run_at, rule_id, table_name, field_name)
);

CREATE INDEX IF NOT EXISTS idx_<table>_run_at ON <history_table> (run_at DESC);
CREATE INDEX IF NOT EXISTS idx_<table>_rule ON <history_table> (rule_id);
```

`UNIQUE (run_at, rule_id, table_name, field_name)` — гарантия идемпотентности.

---

## Workflow в 4 шага

### Шаг 1. Probe & validate

```python
# Проверить что source_table существует и имеет нужные колонки
mcp__postgres__query(sql=f"""
    SELECT column_name FROM information_schema.columns
    WHERE table_schema = '{schema}' AND table_name = '{table}'
""")
# Обязательные: rule_id, rule_type, table_name, total_rows, violations, run_at
```

### Шаг 2. Создание `history_table` если её нет

```python
# postgres-mcp в readonly не позволяет CREATE TABLE — выполняем
# через psql от пользователя или предупреждаем:
print(f"""
CREATE TABLE IF NOT EXISTS {history_table} (
    snapshot_id   bigserial PRIMARY KEY,
    archived_at   timestamptz DEFAULT now(),
    run_at        timestamptz NOT NULL,
    rule_id       text NOT NULL,
    rule_type     text NOT NULL,
    table_name    text NOT NULL,
    field_name    text,
    total_rows    bigint NOT NULL,
    violations    bigint NOT NULL,
    violation_pct numeric(5,2) GENERATED ALWAYS AS
        (CASE WHEN total_rows > 0 THEN 100.0 * violations / total_rows ELSE 0 END) STORED,
    severity      text,
    UNIQUE (run_at, rule_id, table_name, field_name)
);
""")
# Пользователь должен выполнить руками если postgres-mcp readonly.
```

### Шаг 3. Копирование строк (idempotent)

```sql
INSERT INTO <history_table> (run_at, rule_id, rule_type, table_name, field_name,
                             total_rows, violations, severity)
SELECT run_at, rule_id, rule_type, table_name, field_name,
       total_rows, violations, COALESCE(severity, 'MEDIUM')
FROM <source_table>
WHERE run_at = (SELECT MAX(run_at) FROM <source_table>)
ON CONFLICT (run_at, rule_id, table_name, field_name) DO NOTHING;
```

### Шаг 4. Retention — удалить старше `retention_days`

```sql
DELETE FROM <history_table>
WHERE run_at < NOW() - INTERVAL '<retention_days> days';
```

---

## Output Format

```markdown
# DQ Snapshot History

## Архивирование
- Source: {source_table}
- History: {history_table}
- Run timestamp: {run_at}
- Rows archived: {N}
- Total in history: {total_rows} ({distinct_runs} runs)
- Retention: {retention_days} days
- Oldest: {oldest_run_at}

## Что дальше
- Trend будет в `/create-dq-dashboard` (раздел `dq-trend`).
- Для регулярной историзации — добавь в cron.
```

---

## Best Practices

1. **Запускай после `/dq-profiler`** — иначе нечего архивировать.
2. **Cron раз в сутки** в нерабочее время (например `0 2 * * *`):
   ```cron
   0 2 * * *  /usr/bin/claude run /dq-profiler --table=mos.contractors
   5 2 * * *  /usr/bin/claude run /dq-snapshot-history --source=mos.dq_snapshots
   ```
3. **Retention 365 дней** — стандарт для DQ-trend; за год видна сезонность.
4. **NOT перезаписывает** существующие run_at — повторный запуск идемпотентен.
5. **Не используй на live-таблицах с большим объёмом** (>10M) — добавь WHERE по recent run_at явно.

---

## Граничные случаи

| Кейс | Поведение |
|---|---|
| `source_table` пуста | Skip с warning |
| `history_table` не существует | Печатает DDL, ждёт ручного выполнения |
| `run_at` уже в history | Skip (ON CONFLICT DO NOTHING) |
| postgres-mcp в readonly | Печатает SQL, не выполняет — пользователь сам делает |
| `retention_days = 0` | Не удаляет ничего (хранение бесконечно) |
| Изменилась схема `source` | Использует только обязательные колонки, остальное игнорирует |

---

## Связанные

- `/dq-profiler` — генерация snapshots.
- `/create-dq-dashboard` — потребитель history для Trend.
- `/dq-rules-catalog` — каталог правил (опционально для документации).
