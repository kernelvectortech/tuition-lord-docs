# Tuition Lord — Database Schema

> SQLite

---

## Tables

### 1. `student`

```sql
CREATE TABLE student (
    id                TEXT    PRIMARY KEY,
    name              TEXT    NOT NULL,
    monthly_rate      INTEGER NOT NULL,            
    cycle_length_days INTEGER NOT NULL DEFAULT 12,
    guardian_phone    TEXT,                         -- nullable
    address           TEXT,                         -- nullable
    subject           TEXT,                         -- nullable
    schedule_days     TEXT,                         -- nullable, comma-sep weekdays e.g. "MON,WED,FRI"
    schedule_time     TEXT,                         -- nullable, local time HH:mm
    schedule_duration INTEGER,                      -- nullable, minutes
    created_at        TEXT    NOT NULL,             -- ISO-8601 date
    is_archived       INTEGER NOT NULL DEFAULT 0   -- 0 = active, 1 = archived (soft delete)
);
```

---

### 2. `cycle`

```sql
CREATE TABLE cycle (
    id          TEXT    PRIMARY KEY,
    student_id  TEXT    NOT NULL REFERENCES student(id),
    idx         INTEGER NOT NULL,                  -- 1-based per student ("index" is a reserved word)
    start_date  TEXT    NOT NULL,                   -- ISO-8601 local date
    target_days INTEGER NOT NULL,                   -- snapshotted from student.cycle_length_days at creation
    status      TEXT    NOT NULL DEFAULT 'ACTIVE'
                        CHECK (status IN ('ACTIVE', 'SETTLED')),
    settled_at  TEXT,                               -- nullable, ISO-8601 date

    UNIQUE (student_id, idx)                        -- one index value per student
);

CREATE INDEX idx_cycle_student ON cycle(student_id);
```

---

### 3. `class_day`

```sql
CREATE TABLE class_day (
    id          TEXT     PRIMARY KEY,
    student_id  TEXT     NOT NULL REFERENCES student(id),
    cycle_id    TEXT     NOT NULL REFERENCES cycle(id),
    date        TEXT     NOT NULL,                  -- ISO-8601 local calendar date
    status      TEXT     NOT NULL DEFAULT 'SCHEDULED'
                         CHECK (status IN ('SCHEDULED', 'HELD', 'NOT_HELD')),
    source      TEXT     NOT NULL
                         CHECK (source IN ('markToday', 'yesterday', 'tomorrow', 'calendar')),
    note        TEXT,                               -- nullable
    created_at  TEXT     NOT NULL,                  -- ISO-8601 datetime

    UNIQUE (student_id, date)                       -- prevents double-counting on the same day
);

CREATE INDEX idx_class_day_cycle ON class_day(cycle_id);
CREATE INDEX idx_class_day_student ON class_day(student_id);
```

---

### 4. `settlement`

```sql
CREATE TABLE settlement (
    id           TEXT     PRIMARY KEY,
    student_id   TEXT     NOT NULL REFERENCES student(id),
    cycle_id     TEXT     NOT NULL UNIQUE REFERENCES cycle(id),  -- at most one settlement per cycle
    type         TEXT     NOT NULL
                          CHECK (type IN ('FULL', 'PARTIAL')),
    days_counted INTEGER  NOT NULL,
    amount       INTEGER  NOT NULL,                 -- stored in minor units
    payment      TEXT     NOT NULL DEFAULT 'DUE'
                          CHECK (payment IN ('DUE', 'COLLECTED')),
    collected_at TEXT,                               -- nullable, ISO-8601 datetime
    settled_at   TEXT     NOT NULL,                  -- ISO-8601 datetime
    note         TEXT                                -- nullable
);

CREATE INDEX idx_settlement_student ON settlement(student_id);
CREATE INDEX idx_settlement_cycle ON settlement(cycle_id);
```

---

## Design Conventions

| Convention | Detail |
|---|---|
| **Primary keys** | Application-generated UUIDs (`TEXT`). |
| **Dates** | Stored as `TEXT` in ISO-8601 format. Calendar dates use `YYYY-MM-DD`; timestamps use `YYYY-MM-DDTHH:MM:SS`. |
| **Booleans** | SQLite has no native boolean — use `INTEGER` (`0` / `1`). |
| **Enums** | Enforced via `CHECK` constraints on `TEXT` columns. |
| **Soft deletes** | `student.is_archived` flag. Cycles and class days are never hard-deleted (append-only history). |
| **Foreign keys** | Must enable `PRAGMA foreign_keys = ON;` at connection time (SQLite disables them by default). |
| **Unique constraints** | `class_day(student_id, date)` prevents double-counting. `settlement(cycle_id)` enforces one settlement per cycle. `cycle(student_id, idx)` ensures unique cycle indexes per student. |

## Derived Values (Not Stored)

These are always computed at read time from the raw records:

```
perDayRate     = student.monthly_rate / cycle.target_days
daysHeld       = COUNT(class_day WHERE cycle_id = ? AND status = 'HELD')
amountDueSoFar = daysHeld × perDayRate
progress       = daysHeld / target_days    →  "7 / 12"
```
