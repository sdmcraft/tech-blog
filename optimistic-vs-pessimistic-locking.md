# Optimistic vs Pessimistic Locking

## A Story About Trust, Scale, and the Cost of Coordination

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/72d19168-c214-4d90-8ca1-a38e79ce6563" />


Every distributed system eventually runs into the same uncomfortable question:

> What do we assume about conflict?

Not in theory.
Not in an interview.
In production.

Do we assume someone else is about to touch the same piece of data — and block them preemptively?

Or do we assume they probably won’t — and deal with it only if they do?

That quiet assumption is the difference between pessimistic and optimistic locking.

And it turns out, that assumption scales — from a single database row to an entire microservices architecture.

Let’s start small.

---

# The Notebook

Imagine a physical notebook sitting on a desk.

There’s only one copy.

If you need to make changes, you walk over, pick it up, and hold onto it. While you’re writing, nobody else can use it. They wait. When you’re done, you put it back.

Simple. Predictable. Safe.

```
User A  --> [LOCK] --> Write --> Put Back
User B  -------------- WAIT --------------> Write
```

That’s pessimistic locking.

In SQL, it looks like this:

```sql
SELECT * FROM accounts
WHERE id = 101
FOR UPDATE;
```

The database is essentially saying:
“Let’s not take chances.”

You grab the lock first. You do your work. You release it.

No surprises.

But also… no parallelism.

---

# The Shared Document

Now imagine a shared online document.

Ten people open it.
Ten people edit at the same time.

No one blocks anyone.

But when you hit save, the system quietly checks:

“Has someone else changed this since you started?”

If yes, your save fails. You refresh. You merge. You try again.

```
User A --> Edit --> Validate Version --> Save
User B --> Edit --> Validate Version --> Retry (if conflict)
```

That’s optimistic locking.

The database version usually looks like this:

```sql
UPDATE accounts
SET balance = 5000,
    version = version + 1
WHERE id = 101
  AND version = 7;
```

If the version changed underneath you, nothing updates.

No lock. Just detection.

Two different worldviews.

---

# The Real Question

The choice between pessimistic and optimistic locking isn’t really about syntax.

It’s about cost.

Is waiting more expensive than retrying?

In a low-traffic system, waiting doesn’t hurt much.

But in a high-throughput system, waiting spreads.

Threads block.
Connection pools fill.
Latency ripples outward.

Retries, on the other hand, are often cheap.

And that’s why most high-scale systems quietly lean optimistic.

But this is still the easy part.

Because so far, we’re inside a single database.

---

# When We Left the Monolith

There was a time when a “transaction” meant this:

```
BEGIN
  Update Order
  Update Inventory
  Update Payment
COMMIT
```

One database.
One transaction boundary.
One lock manager.

Life was contained.

Then we broke the monolith apart.

Now the flow looks like this:

```
Order Service
      |
      v
Inventory Service
      |
      v
Payment Service
```

Different databases.
Different processes.
Different machines.
Different failure domains.

And now the question of locking becomes far more interesting.

---

# The Temptation of Distributed Pessimism

At first glance, you might think:

“Why not just lock everything?”

Imagine the Order Service acquires a distributed lock before calling Inventory and Payment.

```
Acquire Distributed Lock
        |
        v
Call Inventory
        |
        v
Call Payment
        |
        v
Release Lock
```

Seems safe.

Until Payment times out.

Now the lock is still held.
Retries begin.
Other orders block.
Threads pile up.

What used to be a neat database lock has now turned into a cross-network coordination dependency.

And the network is not reliable.

In distributed systems, holding locks across service boundaries is like holding your breath underwater and hoping nothing unexpected happens.

Something always happens.

---

# The Hidden Cost of Coordination

Here’s the uncomfortable truth:

In distributed systems, coordination is expensive.

Every time multiple services must agree before proceeding, you introduce:

* Latency
* Coupling
* Failure amplification
* Reduced availability

Pessimistic locking is fundamentally about coordination.

And coordination does not scale gracefully.

This is why large-scale systems slowly evolve toward optimism.

Not because it’s ideal.

Because it’s survivable.

---

# The Saga: Optimism Grows Up

Let’s walk through a checkout flow.

A customer places an order.

```
Create Order
      |
      v
Reserve Inventory
      |
      v
Charge Payment
      |
      v
Confirm Order
```

In a monolith, this was one transaction.

In microservices, it’s a sequence of independent steps.

Now imagine Payment fails.

Do we roll back everything atomically?

No.

We compensate.

```
Payment Fails
      |
      v
Release Inventory
      |
      v
Cancel Order
```

We don’t prevent inconsistency.
We repair it.

This is optimistic locking at architectural scale.

We assume things might fail.

And we design recovery paths instead of prevention barriers.

---

# Why This Works Better at Scale

Optimistic distributed systems:

* Avoid long-lived locks
* Avoid global coordination
* Allow services to scale independently
* Tolerate network partitions
* Keep availability high

Yes, they temporarily allow inconsistent states.

But they confine coordination locally and correct globally.

That tradeoff is almost always worth it in internet-scale systems.

---

# A Subtle Shift in Philosophy

Notice what happened.

At the database level:

* Pessimistic locking prevents conflict.
* Optimistic locking detects conflict.

At the distributed level:

* Pessimistic coordination enforces global agreement.
* Optimistic coordination tolerates disagreement and reconciles later.

It’s the same philosophical divide — just magnified.

And as systems grow, the cost of coordination grows faster than the cost of retries.

So systems drift toward optimism.

---

# The Final Reflection

Optimistic vs pessimistic locking isn’t a database feature comparison.

It’s a worldview.

Pessimism says:
“Let’s make sure nothing goes wrong.”

Optimism says:
“Something might go wrong — and we’ll handle it.”

The larger your system becomes, the more that second mindset wins.

Because at scale, preventing all conflict is harder than recovering from it.

And architecture, in the end, is about choosing which costs you are willing to pay.
