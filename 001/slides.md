---
marp: true
theme: rust-demo
transition: fade
paginate: true
html: true
---

<!-- _class: lead -->

# Exploring TigerBeetle

## Debit/Credit Transactions in Conventional Databases vs. First-Class Primitives

### Part of the ongoing Designing Ultra Large Scale Systems study group

---

<!-- _class: build -->

## Who am I?

[**Craig Rodrigues**](https://www.linkedin.com/in/rodrigc)  
Software Engineer in Silicon Valley

- Interested in distributed systems
- Interested in studying interesting topics in this space
- Building a community of like-minded people where we can study together and learn

---

<!-- _class: build -->

## Why This Study Group?

I was inspired to start this study group after taking [Chiradip Mandal](https://www.linkedin.com/in/chiradip/)'s course:

[Data Algorithms](https://chiradip.com/courses/data-algorithms/)

Mastering this topic requires constant study and review of:

- Papers
- Algorithms
- Cutting-edge implementations

---

<!-- _class: build -->

## Guiding Principles

We choose presentations that:

1. **No product pitches** — substance over sales
2. **Adhere to computer science fundamentals** — grounded in first principles
3. **Interesting claims must be verifiable** — papers, code, benchmarks, not vibes
4. **Solve a specific problem that can help the industry at large** — transferable lessons, not navel-gazing

---

<!-- _class: build -->

## Study Group — and Tech Evaluation & Critique

We are a **study group**: we learn together from papers, systems, and implementations.

We also act as a **tech evaluation and critique** group:

- Examine claims carefully — architecture, performance, correctness
- Ask hard questions: what works, what doesn’t, under what assumptions?
- Separate marketing from engineering substance
- Leave with a clearer judgment of when a technology is (or isn’t) a fit

---

<!-- _class: build -->

## Study Group Structure

- Before the meeting: participants are encouraged to study the prerequisite material
- During the event: first 30 minutes presentation, next ~20 minutes open discussion
- Follow-up discussion on the Designing Ultra Large Scale Systems Discord

---

<!-- _class: build -->

## Pre-work Materials

1. [Read: Kyle Kingsbury's analysis of TigerBeetle](https://jepsen.io/analyses/tigerbeetle-0.16.11)
2. [Read: Joran Greef's blog post](https://tigerbeetle.com/blog/2024-07-23-rediscovering-transaction-processing-from-history-and-first-principles/)
3. [Review: TigerBeetle system architecture docs](https://docs.tigerbeetle.com/coding/system-architecture/)
4. [Read: Jim Gray's paper "A Measure of Transaction Processing Power"](https://jimgray.azurewebsites.net/papers/AMeasureOfTransactionProcessingPower.pdf) (defines the classic debit/credit benchmark)
5. [Watch: "1000x: The Power of an Interface for Performance" by Joran Dirk Greef (~1hr)](https://www.youtube.com/watch?v=yKgfk8lTQuE) (Joran discusses the SQL implementation and this paper)

---

<!-- _class: build -->

## [TigerBeetle](https://tigerbeetle.com)

- A financial transactions database purpose-built for high-performance OLTP
- Reimagines debit/credit as a **first-class primitive** (not just SQL queries and locks)
- Aims for massive gains in correctness, safety, and speed at scale

---

<!-- _class: build -->

## [TigerBeetle](https://tigerbeetle.com)

- Strict consistency with double-entry bookkeeping built-in
- Batch thousands of transfers per query; eliminates traditional contention bottlenecks
- Designed from first principles: durability, multi-cloud availability, extreme throughput

---

<!-- _class: build -->

## What We'll Cover

- What a debit/credit is in a financial system
- How debit/credit is implemented in traditional SQL (as defined in Jim Gray's paper [A Measure of Transaction Processing Power](https://jimgray.azurewebsites.net/papers/AMeasureOfTransactionProcessingPower.pdf))
- Existing problems and scalability limitations
- Comparison with TigerBeetle's implementation

---

<!-- _class: diagram -->

## What is a Debit / Credit?

<video src="assets/debit-credit-transfer.webm" autoplay loop muted playsinline></video>

<div class="diagram-caption">

**Debit** − Account A $100 → $99 &nbsp;→&nbsp; **Credit** + Account B $50 → $51

</div>

---

<!-- _class: build -->

## DebitCredit Benchmark

Jim Gray, [*A Measure of Transaction Processing Power*](https://jimgray.azurewebsites.net/papers/AMeasureOfTransactionProcessingPower.pdf) (1985)

The canonical OLTP workload and measure of "transactions per second".

**Sample SQL implementation — debit:**

```sql
BEGIN TRANSACTION;

-- Debit one account
UPDATE accounts
SET balance = balance - :amount
WHERE id = :from_id;
```

---

<!-- _class: build -->

## DebitCredit Benchmark

**Sample SQL implementation — credit & history:**

```sql
-- Credit another account
UPDATE accounts
SET balance = balance + :amount
WHERE id = :to_id;

-- Record history (double-entry)
INSERT INTO history (from_id, to_id, amount, ts)
VALUES (:from_id, :to_id, :amount, NOW());

COMMIT;
```

---

<!-- _class: lead -->

## What's Wrong With This Picture?

### Issues Joran Dirk Greef highlights for SQL debit/credit

[1000x: The Power of an Interface for Performance](https://www.youtube.com/watch?v=yKgfk8lTQuE)  
[Rediscovering Transaction Processing From History and First Principles](https://tigerbeetle.com/blog/2024-07-23-rediscovering-transaction-processing-from-history-and-first-principles/)

---

<!-- _class: build -->

## Impedance Mismatch

One **financial** transfer ≠ one database round-trip.

- For each debit/credit, a general-purpose DB often runs **10–20 SQL queries**
- Data moves to the app over the network; the app decides; results are written back
- Balance checks, updates, and history inserts are separate statements — not one primitive

---

<!-- _class: build -->

## Locks × Network RTT

Concurrency control collides with distributed reality.

- Row locks are held **while waiting on the network**
- Even a few RTTs per transfer cap how many transfers can touch the same rows per second
- More hardware does not fix this — the limit is structural, not just “CPU is too slow”

---

<!-- _class: build -->

## Hot Rows Serialize the System

DebitCredit is contention-heavy by design.

- Many customer accounts debit into a few **hot** internal accounts (e.g. branch / liquidity)
- Hot rows are locked by almost every transfer → **effective serialization**
- Horizontal scaling of the app tier does not remove single-row bottlenecks

---

<!-- _class: build -->

## Correctness Lives Outside the Database

Business rules are reimplemented in application code.

- “Enough balance?”, pending funds, timeouts, double-entry invariants → **app-layer**
- Easy to get wrong under retries, partial failures, and concurrent transfers
- The DB guarantees ACID for SQL statements — not debit/credit **as a domain**

---

<!-- _class: build -->

## The World Got More Transactional

General-purpose OLTP designs are decades old; demand is not.

- Instant payments, gaming economies, energy, per-second cloud billing, …
- Orders of magnitude more transfers — same fundamental row-lock + RTT model
- Joran’s thesis: fix the **interface** (debit/credit as a first-class primitive), not only the engine

---

<!-- _class: lead -->

## TigerBeetle’s Answer

### Not a better Postgres — a **different kind** of database

Debit/credit is the product, not an application pattern on top of SQL.

---

<!-- _class: build -->

## Specialize the Database

TigerBeetle is **not** a general-purpose OLTP store.

- No SQL, no tables for arbitrary schemas, no “store anything”
- Purpose-built for **accounts**, **transfers**, and balances
- Designed to sit **beside** a general-purpose DB (Postgres, etc.), not replace it

See: [System architecture](https://docs.tigerbeetle.com/coding/system-architecture/)

---

<!-- _class: build -->

## Debit / Credit as a First-Class Primitive

The interface *is* the domain model.

| General-purpose SQL | TigerBeetle |
| --- | --- |
| `UPDATE` + `INSERT` + app logic | **Transfer** between two accounts |
| Balance is a column you mutate | Balance is **derived / enforced** by the engine |
| Double-entry is a convention | Double-entry is **built in** |

One API call ≈ one business transfer — not a pile of SQL statements.

---

<!-- _class: build -->

## DebitCredit in TigerBeetle

Same idea as Jim Gray’s DebitCredit — but as a **single transfer primitive**, not multi-statement SQL.

**Create accounts** (once):

```javascript
client.createAccounts([
  { id: 1n, ledger: 1, code: 1, flags: 0 }, // Account A
  { id: 2n, ledger: 1, code: 1, flags: 0 }, // Account B
]);
```

---

<!-- _class: build -->

## DebitCredit in TigerBeetle

**Transfer $1** from Account A → Account B (one request):

```javascript
client.createTransfers([{
  id: id(),                  // idempotency key
  debit_account_id: 1n,      // Account A (from)
  credit_account_id: 2n,     // Account B (to)
  amount: 1n,                // $1 (or 100n for cents)
  ledger: 1,
  code: 1,                   // app-defined reason
}]);
```

No separate `UPDATE` balance, no history `INSERT` — the engine debits, credits, and records the transfer atomically.

---

<!-- _class: build -->

## Fix the Impedance Mismatch

Move the work **into** the database boundary.

- Pack **thousands** of transfers in a single query (batching)
- Amortize network RTTs: many financial txs per round-trip
- Application submits intent; engine applies debit/credit atomically

Before: ~10–20 SQL queries per transfer  
After: up to **thousands of transfers per query**

---

<!-- _class: build -->

## Eliminate the Row-Lock Trap

Specialization enables a different execution model.

- No classic “hold row locks while the app thinks over the network”
- Engine processes a batch of transfers in one place, with **strict serializability**
- Hot accounts still exist — but the design is built for that contention pattern

Performance comes from the **interface**, not only from faster disks.

---

<!-- _class: build -->

## Correctness Inside the Engine

Domain rules are features, not app boilerplate.

- Double-entry bookkeeping enforced by the database
- Balance limits / insufficient funds rejected by the engine
- Pending transfers, timeouts, post/void — financial workflow as primitives

Fewer places for “almost correct” money code in the application.

---

<!-- _class: build -->

## Split the Planes

Use the right tool for each job.

| Hot path (data plane) | Cold path (control plane) |
| --- | --- |
| **TigerBeetle** — balances & transfers | **OLGP** (e.g. Postgres) — users, names, metadata |
| High TPS, strict financial invariants | Flexible schema, strings, reporting joins |

TigerBeetle records *who moved how much*; the general-purpose DB stores *who they are*.

---

<!-- _class: build -->

## The Idea in One Line

> Don’t model debit/credit **in** a general-purpose database.  
> Make a database whose **core abstraction** *is* debit/credit.

That is how TigerBeetle attacks impedance mismatch, lock×RTT limits, and domain correctness at once.

---

<!-- _class: build -->

## Acknowledgments

Thank you:

- **Chiradip Mandal** — https://chiradip.com  
  Inspiration for this study group and for deep study of data algorithms
- **Joran Dirk Greef** and the TigerBeetle team — https://tigerbeetle.com  
  Ideas, writings, and talks on debit/credit as a first-class primitive
- **Kyle Kingsbury** — https://jepsen.io  
  Independent analysis that helps us evaluate systems carefully
