# 🎟️ High-Scale Ticket Booking System — Zero Double Bookings


<img width="1080" height="1350" alt="district_booking_flow" src="https://github.com/user-attachments/assets/a9c69f91-3947-45f4-8474-f291b4e74ce8" />

> **How does a platform like District sell 5,00,000 tickets in 5 minutes without ever double-booking a seat?**

This repo explains the architecture pattern used by large-scale ticketing platforms (District, BookMyShow, Ticketmaster) to handle massive concurrent booking spikes while guaranteeing that **no two users can ever own the same seat**.


---

## 📌 The Problem

- 5,00,000 users hit **"Book Now"** at the same second
- Thousands of them tap the **same seat** within milliseconds
- A naive `SELECT ... then UPDATE` creates a classic **race condition**
- Row-level DB locking melts under this load — the database becomes the bottleneck

**Goal:** Correctness (no double booking) + Fairness (FIFO) + Speed (sub-50ms booking path)

---

## 🏗️ Architecture Overview

```
Users ──► Virtual Waiting Room ──► Seat Map Service ──► Atomic Seat Lock (Redis)
                                                              │
                                              ┌───────────────┴───────────────┐
                                         Lock acquired ✅              Lock failed ❌
                                              │                  "Seat just got taken"
                                              ▼
                                        Payment Window (7 min TTL)
                                              │
                                  ┌───────────┴───────────┐
                              Paid ✅                  Failed / abandoned ❌
                                  │                    TTL expires → seat auto-released
                                  ▼
                        DB Write — UNIQUE(event_id, seat_id)
                                  │
                                  ▼
                        Kafka → email / SMS / invoice / analytics
```

---

## 1️⃣ Virtual Waiting Room

All 5 lakh users **never** hit the backend simultaneously. A queue (token-bucket / Queue-it style gate) admits users in fair **FIFO batches**.

- Kills the thundering herd before it reaches your services
- Provides fairness — first come, first served
- Backend only ever sees a sustainable request rate

---

## 2️⃣ Seat Map Service (Reads from Redis)

Real-time seat availability is served entirely from **Redis** — the database is never hit for reads during peak. Sold/held/available states live in fast in-memory structures.

---

## 3️⃣ Atomic Seat Lock — The Real Hero ⭐

When a user taps seat `A12`:

```redis
SET seat:EVT123:A12 user_456 NX EX 420
```

### Key anatomy

```
seat:EVT123:A12
  │      │     └── seat number
  │      └──────── event ID (Saturday's show ≠ Sunday's show)
  └─────────────── namespace prefix
```

The **event ID** in the key makes each `(event, seat)` pair independent — locking A12 for one show doesn't block A12 for another.

### Why this can never double-book

| Flag | Meaning |
|------|---------|
| `NX` | Set **only if the key does Not eXist** |
| `EX 420` | Auto-expire after 420 seconds (7 minutes) |

Redis executes commands on a **single thread** — the check-and-set is one indivisible operation. Two users can **never** both win:

```
10:00:00  User A → SET ... NX EX 420  →  OK ✅   (key created, seat held)
10:00:02  User B → SET ... NX EX 420  →  nil ❌  ("Seat just got taken")
```

### What if the booking is never completed?

Nothing needs to "happen" — the TTL handles it:

```
10:07:00  User A never paid → Redis auto-deletes the key (TTL expired)
10:07:05  User C → SET ... NX EX 420  →  OK ✅  (key gone, NX succeeds again)
```

**No cron jobs. No cleanup workers. No stuck inventory.** A non-existent key is exactly the condition `NX` checks for, so expiry automatically makes the seat bookable again.

---

## 4️⃣ Safe Lock Release (the subtle bug everyone misses)

If a user cancels manually, you must **delete only your own lock** — never blindly `DEL`:

```
10:07:00  User A's lock expires (TTL)
10:07:05  User C acquires the seat
10:07:06  User A's stale "cancel" request arrives → blind DEL would delete C's lock! 🐛
```

Fix: atomic **check-and-delete** via Lua:

```lua
-- Release the lock only if WE own it
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```
KEYS[1] = seat:EVT123:A12
ARGV[1] = user_456   -- must match the lock value
```

For **multi-seat bookings** (4 seats together), a Lua script locks all seats atomically — **all or none** — preventing partial holds and deadlocks.

---

## 5️⃣ Database UNIQUE Constraint — Last Line of Defense

Distributed locks can theoretically fail (Redis failover, clock drift, GC pause past lock expiry — see Kleppmann's critique of Redlock). So the final insert never trusts the lock alone:

```sql
CREATE TABLE bookings (
    id          BIGINT PRIMARY KEY,
    event_id    VARCHAR(32) NOT NULL,
    seat_id     VARCHAR(16) NOT NULL,
    user_id     BIGINT      NOT NULL,
    created_at  TIMESTAMP   NOT NULL,
    CONSTRAINT uq_event_seat UNIQUE (event_id, seat_id)
);
```

Even if two requests both *believe* they hold the lock, the second `INSERT` throws a constraint violation and is rejected.

> **Correctness is guaranteed by the database. Performance is guaranteed by Redis.**

Plus: **idempotency keys** on the payment path, so gateway retries never create duplicate bookings.

---

## 6️⃣ Kafka — Async Fan-out

Everything non-critical leaves the hot path:

- Confirmation email / SMS
- Invoice generation
- Analytics & seat-map cache invalidation

The synchronous booking path stays lean (~50ms).

---

## 🔥 Failure Scenario: Redis Crashes Mid-Sale

**Q: Redis dies with 50,000 active seat locks. Now what?**

- Redis runs with **replication + AOF persistence**; a replica is promoted within seconds
- Worst case: some in-flight locks are lost during failover — a held seat may briefly *appear* available
- That failure mode can only cause a **constraint violation at checkout** for a few users — never two people owning the same seat

The system is **fail-safe, not fail-silent**: it degrades to "please pick another seat," never to corrupted bookings.

---

## 🧠 Key Takeaways

| Property | Achieved by |
|----------|-------------|
| **Correctness** | Atomicity — `SET NX` + DB `UNIQUE` constraint |
| **Fairness** | Queueing — virtual waiting room (FIFO) |
| **Speed** | Redis in the hot path, DB out of it, Kafka for async |
| **Self-healing inventory** | TTL-based auto-release, no cleanup jobs |

---

## 🛠️ Tech Stack Referenced

`Redis` · `Lua scripting` · `PostgreSQL / MySQL` · `Kafka` · `Java / Spring Boot`

---

## 🤝 Connect

If this helped you think about system design differently, ⭐ the repo and connect with me:

- **LinkedIn:** [linkedin.com/in/kgstrivers](https://linkedin.com/in/kgstrivers)
- **Medium:** [kgstrivers.medium.com](https://kgstrivers.medium.com)
- **GitHub:** [github.com/kaushikpuka1998](https://github.com/kaushikpuka1998)
