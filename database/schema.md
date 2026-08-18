# Tuition Lord — Database Schema

> SQLite

---

## Tables

### 1. `student`

```sql
CREATE TABLE student (
    id                      TEXT    PRIMARY KEY,
    name                    TEXT    NOT NULL,
    monthly_rate            INTEGER NOT NULL,             -- stored in minor units (e.g., poisha)
    cycle_length_sessions   INTEGER NOT NULL DEFAULT 12,
    student_phone           TEXT,                         -- nullable
    guardian_name           TEXT,                         -- nullable
    guardian_phone          TEXT,                         -- nullable
    address                 TEXT,                         -- nullable
    subject                 TEXT,                         -- nullable
    created_at              TEXT    NOT NULL,             -- ISO-8601 datetime
    updated_at              TEXT    NOT NULL,             -- ISO-8601 datetime
    is_archived             INTEGER NOT NULL DEFAULT 0    -- 0 = active, 1 = archived (soft delete)
);
```

---

### 2. `cycle`

```sql
CREATE TABLE cycle (
    id               TEXT    PRIMARY KEY,
    student_id       TEXT    NOT NULL REFERENCES student(id),
    idx              INTEGER NOT NULL,                  -- 1-based per student ("index" is a reserved word)
    start_date       TEXT    NOT NULL,                  -- ISO-8601 local date
    target_sessions  INTEGER NOT NULL,                  -- snapshotted from student.cycle_length_sessions at creation
    status           TEXT    NOT NULL DEFAULT 'ACTIVE'
                             CHECK (status IN ('ACTIVE', 'SETTLED')),
    settled_at       TEXT,                              -- nullable, ISO-8601 datetime
    updated_at       TEXT    NOT NULL,                  -- ISO-8601 datetime

    UNIQUE (student_id, idx)                            -- one index value per student
);

CREATE INDEX idx_cycle_student ON cycle(student_id);
```

---

### 3. `class_day`

```sql
CREATE TABLE class_day (
    id          TEXT     PRIMARY KEY,
    student_id  TEXT     NOT NULL REFERENCES student(id),
    cycle_id    TEXT              REFERENCES cycle(id),     -- NULLABLE: Assigned only when day resolves to HELD/NOT_HELD
    date        TEXT     NOT NULL,                          -- ISO-8601 local calendar date
    slot        INTEGER  NOT NULL DEFAULT 1,                -- 1-based session index within the same day (allows multiple sessions)
    status      TEXT     NOT NULL DEFAULT 'SCHEDULED'
                         CHECK (status IN ('SCHEDULED', 'HELD', 'NOT_HELD')),
    source      TEXT     NOT NULL
                         CHECK (source IN ('markToday', 'yesterday', 'tomorrow', 'calendar', 'fromReminder')),
    note        TEXT,                                       -- nullable
    created_at  TEXT     NOT NULL,                          -- ISO-8601 datetime
    updated_at  TEXT     NOT NULL,                          -- ISO-8601 datetime (audit trail for status changes)

    UNIQUE (student_id, date, slot)                         -- allows multiple sessions per day; slot disambiguates them
);

CREATE INDEX idx_class_day_cycle ON class_day(cycle_id);
CREATE INDEX idx_class_day_student ON class_day(student_id);
```

---

### 4. `student_schedule`

```sql
CREATE TABLE student_schedule (
    id          TEXT    PRIMARY KEY,
    student_id  TEXT    NOT NULL REFERENCES student(id),
    day_of_week TEXT    NOT NULL
                        CHECK (day_of_week IN ('MON','TUE','WED','THU','FRI','SAT','SUN')),
    start_time  TEXT    NOT NULL,                           -- local time HH:mm
    duration    INTEGER NOT NULL,                           -- minutes
    updated_at  TEXT    NOT NULL,                           -- ISO-8601 datetime

    UNIQUE (student_id, day_of_week)                        -- one time-slot per weekday per student
);

CREATE INDEX idx_student_schedule_student ON student_schedule(student_id);
```

---

### 5. `settlement`

```sql
CREATE TABLE settlement (
    id               TEXT     PRIMARY KEY,
    student_id       TEXT     NOT NULL REFERENCES student(id),
    cycle_id         TEXT     NOT NULL UNIQUE REFERENCES cycle(id),  -- at most one settlement per cycle
    sessions_counted INTEGER  NOT NULL,
    amount           INTEGER  NOT NULL,                 -- stored in minor units (poisha)
    payment_status   TEXT     NOT NULL DEFAULT 'DUE'
                              CHECK (payment_status IN ('DUE', 'COLLECTED', 'PARTIALLY_COLLECTED')),
    collected_at     TEXT,                              -- nullable, ISO-8601 datetime
    settled_at       TEXT     NOT NULL,                 -- ISO-8601 datetime
    updated_at       TEXT     NOT NULL,                 -- ISO-8601 datetime
    note             TEXT                               -- nullable
);

CREATE INDEX idx_settlement_student ON settlement(student_id);
CREATE INDEX idx_settlement_cycle ON settlement(cycle_id);
```

---

## Design Conventions & Accepted Simplifications

| Convention | Detail |
|---|---|
| **Primary keys** | Application-generated UUIDs (`TEXT`). |
| **Money** | Stored as `INTEGER` in minor units (poisha). E.g., ৳3000 is stored as `300000`. Rounded only at the display layer to avoid floating point math errors. |
| **Dates / Timestamps** | Stored as `TEXT` in ISO-8601 format. Calendar dates use `YYYY-MM-DD`; timestamps use `YYYY-MM-DDTHH:MM:SS`. Consistent across all timestamp fields. |
| **Booleans** | SQLite has no native boolean — use `INTEGER` (`0` / `1`). |
| **Enums** | Enforced via `CHECK` constraints on `TEXT` columns. |
| **Soft deletes** | `student.is_archived` flag. Cycles and class days are never hard-deleted (append-only history). |
| **Foreign keys** | Must enable `PRAGMA foreign_keys = ON;` at connection time (SQLite disables them by default). |
| **Unique constraints** | `class_day(student_id, date, slot)` prevents duplicate slots on the same day while allowing multiple sessions. `student_schedule(student_id, day_of_week)` ensures one time-block per weekday per student. `settlement(cycle_id)` enforces one settlement per cycle. `cycle(student_id, idx)` ensures unique cycle indexes per student. |

## Intentional Limitations (Accepted Simplifications)

1. **Floating scheduled days:** Future `SCHEDULED` class days deliberately have a `NULL` `cycle_id` so they are not marooned if a tutor settles the current cycle early. The `cycle_id` is only attached the moment the status resolves to `HELD` or `NOT_HELD`.

## Derived Values (Not Stored)

These are always computed at read time from the raw records:

```
perSessionRate = student.monthly_rate / cycle.target_sessions
sessionsHeld   = COUNT(class_day WHERE cycle_id = ? AND status = 'HELD')
amountDueSoFar = sessionsHeld × perSessionRate
progress       = sessionsHeld / target_sessions    →  "7 / 12"
```
