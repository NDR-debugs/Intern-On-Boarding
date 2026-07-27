# BookSwap — Storage and Cache Decisions

> **How to read this doc:** it explains *what data we keep*, *where each piece
> lives*, *what we copy into a fast cache*, and *what work we push onto a queue*.
> The golden rule throughout: **the database is the source of truth. The cache
> and the queue are helpers that can be thrown away and rebuilt.**

---

## 1. Data inventory

*What we store, roughly how much, and whether it is read or written more.*

| Data type | Example record | Volume estimate (1 yr) | Read/write ratio |
|-----------|----------------|------------------------|------------------|
| Book listing | one row per book (title, author, ISBN, condition, available) | ~50,000 across all buildings | **read-heavy** (browsed far more than edited) |
| Member profile | one row per person (name + private contact) | ~10,000 | read-heavy |
| Book photo | one JPEG/PNG file, up to 5 MB | ~50,000 files | write-once, read-often |
| Borrow request | pending/accepted/declined for a book | ~100,000 | balanced |
| Loan record | out / returned / overdue, with borrower history | ~120,000 | balanced |
| "Recently added" list | the 10 newest books per building | tiny, per building | **very read-heavy** (digest + home screen) |

---

## 2. Storage selection

*Each data type gets one **primary** store. "Why not the alternative" says why
the obvious other option is worse **for this specific data**.*

| Data type | Chosen store | Why this store | Why not the alternative |
|-----------|--------------|----------------|--------------------------|
| Book listing | **Azure SQL** | Relational; a book has a foreign key to its owner and to loans. Joins ("show me this member's books and their loans") are natural. Filtering/sorting the catalogue is what SQL indexes are built for. | *Cosmos DB (document):* we'd hand-roll joins in app code. *Redis:* not durable — a cache, not a home. |
| Member profile | **Azure SQL** | Small, structured, joined to books and loans. Column-level control makes it easy to **never select** the private address/phone columns in API queries (privacy rule). | *Cosmos DB:* flexible schema we don't need here; harder to guarantee we never leak a private field. |
| Book photo | **Azure Blob Storage** | Binary files up to 5 MB belong in object storage: cheap, scales sideways, CDN-friendly for fast image loads. SQL only stores the **URL**, not the bytes. | *Azure SQL BLOB column:* bloats the database and every backup with megabytes of images; slows queries. Requirement literally says never store photos in the app DB. |
| Borrow request | **Azure SQL** | Short-lived state (pending→accepted/declined) tied by FK to a book and a member. Needs a transaction with the loan it may create. | *Redis:* would lose in-flight requests on a restart. *Queue:* a queue moves work, it doesn't *store* state you query later. |
| Loan record | **Azure SQL** | The system of record for who has what, due dates, and history. Overdue = a simple date query (`dueAt < now AND returnedAt IS NULL`). ACID matters: a book can't be "out" to two people. | *Cosmos DB:* cross-document consistency (book availability + loan) is awkward; SQL gives it in one transaction. |
| "Recently added" list | **Azure Cache for Redis** (copy) | Same 10 rows asked for constantly and they change slowly. Perfect to cache. Source of truth stays in SQL. | *Only in Redis:* no — if Redis is wiped the list must rebuild from SQL, so SQL is still the home. |

**Search (title / author / ISBN, p95 < 300 ms for 5,000 books):** handled by
**Azure SQL** with indexes on the searched columns. 5,000 rows per building is
small; an indexed query is comfortably under 300 ms. Hot search results are
cached in Redis (see below). *A dedicated search engine (Azure AI Search) is
overkill at this size — note it as a "later, if we grow to millions of books"
upgrade, not a day-1 container.*

### Source of truth — one line each

- **Books, members, borrow requests, loans →** Azure SQL.
- **Photo bytes →** Azure Blob Storage (SQL keeps the URL pointer).
- **Redis →** never a source of truth. It is a disposable fast copy.
- **Service Bus (queue) →** never a source of truth. It is a temporary buffer of work.

---

## 3. Cache plan

**What is hot enough to cache?** (rule: *asked far more often than it changes*)

- **Single book metadata** — `book:{id}`. Browsed constantly, edited rarely.
- **"Recently added" list per building** — `building:{id}:recent`. Read on every
  home-screen open and by the weekly digest; changes only when someone lists a book.
- **Popular search results** — `search:{building}:{query}:{page}`. The same few
  searches repeat all day.

**TTL (time-to-live) and why:**

- `book:{id}` → **60 s**. Metadata barely changes; 60 s of staleness is harmless
  and we also delete the key on edit (below), so reads stay fresh.
- `building:{id}:recent` → **300 s** *and* deleted whenever a new book is listed.
- `search:...` → **30 s**. Short, because new books should show up quickly.

**Invalidation strategy:** cache-aside + delete-on-write. When the underlying
row changes we **delete** the cache key so the next read repopulates it from SQL.
We never trust the cache to update itself.

**Cache-aside pattern (pseudocode):**

```js
// READ: look in cache first, fall back to DB, then fill the cache.
async function getBook(bookId) {
  const key = `book:${bookId}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);          // cache HIT

  const row = await db.query("SELECT * FROM books WHERE id = $1", [bookId]);
  if (!row) return null;                           // not in DB at all

  // TTL 60s: book metadata rarely changes, so 60s stale is acceptable.
  await redis.set(key, JSON.stringify(row), "EX", 60);
  return row;                                      // cache MISS -> now warmed
}

// WRITE: update the DB (the truth), then INVALIDATE the stale cache entry.
async function updateBook(bookId, patch) {
  const updated = await db.update("books", bookId, patch);
  await redis.del(`book:${bookId}`);               // next read repopulates
  return updated;
}
```

**When NOT to cache (at least one example):**

- **Book availability / live loan status.** This flips the moment someone borrows
  or returns. A stale "available: true" would let two neighbours request a book
  that's already gone — confusing and unfair. We always read availability straight
  from SQL (or invalidate the key the instant it changes).
- **Private member contact details.** Small, sensitive, and rarely read — caching
  buys nothing and spreads private data to another system. Not worth it.

---

## 4. Queue plan

We use **Azure Service Bus** so slow or optional work never blocks the user.

**What goes on the queue and why:**

- **Weekly digest emails** — best-effort and heavy (fan-out to every member).
  Pushed as messages; a background worker sends them via Azure Communication
  Services. Requirement: *listing a book must succeed even if email is down* — so
  `POST /books` only writes to SQL, returns **201**, and drops a "book added"
  event on the queue. Email being down can never fail a listing.
- **In-app notifications** ("someone wants to borrow your book") — enqueued so the
  API responds instantly; a fast consumer pushes the notification (target: in-app
  within 2 s). Decoupling keeps the borrow request itself snappy.

**Failure mode — consumer down for 30 minutes:**

- Messages **stay in the queue** (Service Bus retains them); nothing is lost.
- When the worker comes back it drains the backlog. Digest emails go out late —
  acceptable, they're best-effort.
- **Retries:** each message is retried with back-off (e.g. 5 attempts).
- **Dead-letter queue (DLQ):** a message that keeps failing (e.g. a malformed
  address) is moved to the DLQ after max retries, so one bad message never blocks
  the rest. We alert on DLQ depth and inspect those messages by hand.

**5xx / retry note (for Day 3):** if a downstream call returns a 5xx, the consumer
retries with exponential back-off + jitter and a cap on attempts; only then does
it dead-letter. Producers (the API) do **not** retry on the user's behalf — they
enqueue once and return, so the user is never left waiting on a flaky dependency.

---

### One-paragraph summary a senior engineer can approve

Everything durable lives in **Azure SQL** (books, members, borrow requests,
loans), because the data is relational and consistency matters — a book can't be
lent twice. **Photos** live in **Blob Storage**; SQL only holds the URL. **Redis**
holds disposable copies of the few hot, slow-changing reads (a book, the
recently-added list, popular searches) with short TTLs and delete-on-write
invalidation; we deliberately **don't** cache live availability or private data.
**Service Bus** absorbs optional/slow work (digest emails, notifications) so a
listing succeeds even when email is down, with retries and a dead-letter queue
for poison messages. SQL and Blob are the source of truth; Redis and Service Bus
are helpers we can wipe and rebuild.
