# Decisions log

## 001: Project domain is EU electricity prices (2026-10-25)

**Decision:** Build the project on EU electricity prices and generation data.

**Alternatives considered:** Dutch railway delays (kept for a second project).

**Why:**
- It teaches time-series data engineering and gives more opportunity to apply dbt and data engineering concepts.
- It is relevant for my job search: energy and climate tech, including EV charging, is hiring heavily in the EU.

## 002: All timestamps are stored in UTC (2026-10-25)

**Decision:** All timestamps are stored in UTC.

**Alternatives considered:** Local time zones.

**Why:**
- UTC avoids ambiguity. On 25 Oct 2026, the Amsterdam clock jumps back from 03:00 to 02:00, so 02:30 happens twice. With local timestamps, we couldn't tell which 02:30 is which.
- On the spring transition, 02:30 doesn't exist at all.
- UTC gives every instant a unique timestamp, which lets me join Dutch, German and French electricity data accurately.

## 003: The pipeline never assumes a fixed number of slots per day (2026-10-25)

**Decision:** The pipeline never assumes a fixed number of slots per day.

**Alternatives considered:** Assuming 96 slots per day.

**Why:**
- curveType `A03` leaves out repeated values, so a day can have fewer points than slots.
- On daylight saving days, the number of slots increases (autumn) or decreases (spring).

**What the pipeline does instead:**
- It knows how many slots to expect on a normal day and on clock-change days.
- It first fills `A03` gaps by carrying the previous price forward.
- If the slot count still doesn't match, it raises an alert.