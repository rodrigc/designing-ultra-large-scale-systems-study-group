---
marp: true
theme: rust-demo
transition: fade
paginate: true
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

<!-- _class: build -->

## DebitCredit Benchmark

Jim Gray, [*A Measure of Transaction Processing Power*](https://jimgray.azurewebsites.net/papers/AMeasureOfTransactionProcessingPower.pdf) (1985)

The canonical OLTP workload and measure of "transactions per second".

**Sample SQL implementation:**

```sql
BEGIN TRANSACTION;

-- Debit one account
UPDATE accounts
SET balance = balance - :amount
WHERE id = :from_id;

-- Credit another account
UPDATE accounts
SET balance = balance + :amount
WHERE id = :to_id;

-- Record history (double-entry)
INSERT INTO history (from_id, to_id, amount, ts)
VALUES (:from_id, :to_id, :amount, NOW());

COMMIT;
```
