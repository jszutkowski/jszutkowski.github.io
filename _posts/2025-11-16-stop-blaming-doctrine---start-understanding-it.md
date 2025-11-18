---
layout: post
title: Stop Blaming Doctrine - Start Understanding It
date: 2025-11-16 12:00:00
categories: [programming]
tags: [php, doctrine, persistence, sql, symfony]     # TAG names should always be lowercase
description: A practical breakdown of how Doctrine really works under the hood — Unit of Work, Identity Map, lazy loading, and the common misconceptions that cause most performance issues. Stop blaming the ORM and start understanding it.
author: jaroslaw_szutkowski
published: true
---

Every few months, someone on the team sighs during a retrospective and says the words that make me reach for coffee:

> "Doctrine is slow."

No, it's not. Doctrine is an ORM. It does exactly what you tell it to do. The problem is - most developers don't realise *how much* they're telling it to do.

This post isn't a tutorial. It's a field report from someone who's watched Doctrine crawl through millions of rows... and then, after a few lines of code change, **fly through them using less memory than Chrome with two tabs open**.

---

## Scene 1: The 900 MB Disaster

A CLI command meant to clean up stale product records.
Half a million entities. Nothing fancy. Just iterate and tweak a few fields.

```php
$products = $this->repository->createQueryBuilder('p')
    ->getQuery()
    ->toIterable();

foreach ($products as $product) {
    $product->deactivate();
    $this->em->flush();
    $this->em->clear();
}
```

Initial memory usage: **45 MB**.
End of run: **~900 MB**.

The process finished - barely. But it shouldn't have taken down the server on its way out.

So what happened?

---

## Scene 2: The Culprit - Buffered Queries

When you call `toIterable()`, Doctrine promises *lazy iteration*. But under the hood, **PHP's `mysqlnd` driver** executes queries in **buffered mode** by default.

That means:

* The entire result set is loaded into PHP memory first.
* Only *after that* does Doctrine start hydrating entities lazily.

In English: *You're still fetching the whole table - just slower.*

> "With `mysqlnd`, PHP's memory limit includes the full result set - even before you touch it."

So while you thought you were streaming, you were actually drinking from a firehose.

---

## Scene 3: The Fix - Stop Hoarding Data

The simplest, production-safe pattern is **chunked fetching**: process data in small slices, clear the EntityManager between them, and let Doctrine breathe.

```php
public function getEntities(): iterable
{
    $id = 0;
    do {
        $qb = $this->createQueryBuilder('p')
            ->andWhere('p.id > :id')
            ->orderBy('p.id', 'ASC')
            ->setMaxResults(1000)
            ->setParameter('id', $id);

        $entities = $qb->getQuery()->toIterable();
        $hasResults = false;

        foreach ($entities as $entity) {
            $hasResults = true;
            $id = $entity->getId();
            yield $entity;
        }
    } while ($hasResults);
}
```

Result:

* Memory flatlines around **90 MB**
* Throughput improves by 40 %
* Database I/O becomes predictable

You didn't need a different ORM. You just needed to stop asking Doctrine to hold 100 000 objects at once.

> ⚙️ **Note:** after each `$em->clear()`, entities are detached. If you rely on cascades or plan to reuse the same object instance later, you'll need to fetch it again. Otherwise, you'll silently lose pending changes.

---

### 💡 BONUS RESOURCE – Free PDF Guide

> ⚙️ Want the quick-reference version of this post?
>
> Grab my free **Doctrine Performance Checklist (PDF)** -
> "10 Steps to a Faster Doctrine-Based App."
>
> You'll get a concise, printable guide that covers:
>
> * how to spot N+1s in seconds,
> * which Doctrine hints actually matter,
> * how to batch process millions of rows safely.
>
> 🔗 [**Download the free guide here**](https://doctrine-performance-list.doctrinelab.pl){:target="_blank"}
> *(no spam, just pure optimization wisdom)*

---

## Scene 4: Going Leaner - IDs First

If you really want to see Doctrine behave like a streaming engine, go one step further: fetch only IDs, and hydrate each entity just before you need it.

```php
foreach ($repo->iterateIdsAsc() as $id) {
    $product = $em->find(Product::class, $id);
    $product->deactivate();

    if (++$count % 100 === 0) {
        $em->flush(); // Doctrine wraps this in its own transaction
        $em->clear(); // release memory
    }
}
```

Doctrine now processes entities in small, predictable batches.
Each loop hydrates only one object at a time and keeps it in memory until the entity manager is cleared. And since each record is fetched on demand, you're always working with the most up-to-date database state.

---

## Scene 5: The Hidden Switch Nobody Talks About - Buffered Queries

If you're performing large **read-only** scans, you can strip Doctrine of almost all overhead - no tricks, no custom SQL.

Doctrine uses PDO under the hood. Add this to your DB connection config:

```php
'driverOptions' => [PDO::MYSQL_ATTR_USE_BUFFERED_QUERY => false],
```

Suddenly, PHP stops hoarding the entire result set in memory and starts streaming rows directly from MySQL. Memory footprint: minimal. Performance: predictable.

> ⚠️ **Safety sidebar - Unbuffered queries bite back:**
>
> * Always consume the full result before starting another query on the same connection, or you'll deadlock the driver.
> * Avoid lazy-loading relations inside loops; it will trigger additional queries before the stream finishes.
> * Unbuffered + large payloads? Watch out for timeouts and dropped connections. Implement retry logic.
>
> Handle them right, and unbuffered mode becomes your best friend for analytics-scale reads.

---

## Scene 6: The Hidden Switch Nobody Talks About - `HINT_READ_ONLY`

The second overlooked optimization is telling Doctrine itself to stop tracking entities.

```php
$query->setHint(\Doctrine\ORM\Query::HINT_READ_ONLY, true);
```

With that hint, Doctrine hydrates entities but doesn't register them in the Unit of Work - meaning no change tracking, no dirty checks. It's important to understand that `HINT_READ_ONLY` **does not skip hydration** - entities are still created, they're just not managed. If you want zero hydration, use DBAL's `iterateAssociative()`.

You didn't touch a single entity. You just told Doctrine to stop worrying.

---

## Scene 7: Why People Think Doctrine Is Slow

Because they don't understand **what it's doing for them**.

Doctrine is like an F1 car in traffic. If you floor it in first gear, it will scream and overheat. But once you learn *how it shifts*, it'll lap your hand-rolled SQL scripts without breaking a sweat.

The ORM:

* Hydrates entities lazily when asked.
* Tracks changes only if you let it.
* Wraps `flush()` in its own transaction - you don't need to micromanage that.
* Lets you drop down to SQL when business logic allows.

So when someone says *"Doctrine can't handle large data sets"*, what they really mean is *"I made Doctrine do something stupid."*

---

## Scene 8: The Trade-offs You Should Know

| Goal            | Pattern                             | Why it works                                      |
| --------------- | ----------------------------------- | ------------------------------------------------- |
| Minimal memory  | IDs-first iteration                 | One entity hydrated at a time                     |
| Balanced speed  | Chunked keyset pagination + clear() | Keeps DB cursor hot and resets UoW per chunk      |
| Consistency     | Snapshot IDs once                   | Deterministic and repeatable batch re-runs        |
| Bulk writes     | Raw DQL or SQL                      | ORM overhead is **negligible** in practice        |
| Streaming reads | DBAL `iterateAssociative()`         | No entity hydration at all - raw arrays, zero UoW |

---

## Scene 9: Measure, Don't Assume

Want proof? Take `memory_get_peak_usage(true)` at checkpoints and log it.

If your memory curve looks like a staircase, you're holding too many entities. If it's a flat line, you've nailed it.

Doctrine isn't magic - but it's measurable.

```php
$start = microtime(true);
// ... your batch logic ...
$elapsed = microtime(true) - $start;
$peak = memory_get_peak_usage(true) / 1024 / 1024;
printf("Processed %d records in %.2fs (%.1f MB peak)\n", $count, $elapsed, $peak);
```

> And for the love of your ops team - disable SQL logging in batch jobs.

---

## Scene 10: Lessons From Production

In one of our production pipelines, we process **massive data volumes** daily using Doctrine - comfortably, predictably, and with modest memory use.

Here's the operational frame:

* Batch size: around 1 000 records per chunk
* Keyset: `(id)` based pagination
* Database: MySQL 8
* Memory footprint: roughly 90 MB during steady-state processing

The secret wasn't a custom ORM. It was discipline:

* `flush()` and `clear()` in predictable batches
* Use `HINT_READ_ONLY` for data that doesn't need tracking
* Drop down to DBAL or use `SELECT NEW DTO` statements when you just need raw rows

```php
$query = $em->createQuery('SELECT NEW App\\DTO\\ProductReport(p.id, p.name, p.price) FROM App\\Entity\\Product p');
foreach ($query->toIterable() as $dto) {
    // lightweight read
}
```

* Avoid lazy-loading inside loops
* Monitor memory and runtime metrics per batch

The bottleneck was never Doctrine.

---

## Scene 11: Anti-Patterns I Keep Seeing in Production

Some of these will sound familiar. They're the patterns that quietly turn batch jobs into memory leaks:

1. Calling `findAll()` or `->getResult()` on a 3-million-row table.
2. Running `flush()` *on every iteration*.
3. Triggering lazy-loading in loops (N+1 queries multiplied by millions).
4. Using offset-based pagination: `OFFSET 10M LIMIT 1k`.
5. Keeping the SQL logger enabled in background jobs.

> **Quick sanity checklist:**
> ☑️ No offset-based pagination
> ☑️ Predictable `flush()` cadence
> ☑️ No implicit lazy loads inside loops
> ☑️ SQL logger turned off

A single screenshot from your profiler after applying these rules is worth a thousand words.

---

## Scene 12: Operational BHP for Batch Jobs

This is the stuff that separates scripts that *work locally* from ones that *survive nightly runs in production*:

* **Idempotency:** snapshot IDs or mark rows (`status`, `processed_at`).
* **Retry and resume:** pick up where you left off (based on the last processed ID).
* **Backpressure:** limit concurrency to avoid choking replicas.
* **Timeouts and deferred constraints:** use them wisely depending on your database.

It's the difference between *code that runs* and *code that runs reliably*.

---

## Scene 13: What It All Means

Doctrine is a precision instrument. When used carelessly, it can make a mess; when handled with intent, it performs beautifully.

Learn how it hydrates, buffers, and tracks changes - and use each feature for the job it's meant for. Once you do, Doctrine will handle even heavy workloads calmly and efficiently.

It's not slow. It just rewards developers who understand how it works.

---

### 🚀 Ready to Go Beyond Quick Fixes?

If you liked this article, you'll love my upcoming course -
**Doctrine Efficiency Lab** 💻

Learn how to:

* handle millions of records without custom SQL,
* debug ORM performance in real time,
* and stop Doctrine from being "that slow layer" in your stack.

🎁 When you join the waitlist, you'll immediately get
**my free 10-Step Doctrine Optimization PDF** -
a condensed version of everything we discussed here.

🔗 [**Join the waitlist + get the guide**](https://doctrine-performance-list.doctrinelab.pl){:target="_blank"}

