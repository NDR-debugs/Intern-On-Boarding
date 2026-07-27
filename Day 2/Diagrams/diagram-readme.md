# Container Diagram — README

**Files:** `container-diagram.png` (the picture) · `container-diagram.drawio`
(editable source — open at [draw.io](https://app.diagrams.net)).

**What it is:** a **C4 Level 2 (Container) diagram**. In C4, a "container" isn't
Docker — it means a separately runnable/deployable thing (an app, a database, a
queue). Level 2 shows the **big building blocks and how they talk**, without going
inside any one of them.

---

## How to read it (in 5 sentences)

1. Start at the top: a **Member** (person) only ever touches the **Mobile app** —
   never the API directly.
2. The mobile app first signs in with **Identity (Entra)** to get a JWT, then makes
   **REST calls** to the **API service**, which is the brain of the system.
3. Follow the **solid black arrows** for synchronous work — the API reads/writes the
   **Azure SQL** database (the source of truth), checks the **Redis** cache for hot
   reads, and stores/serves photos from **Blob Storage**.
4. Follow the **dashed orange arrows** for asynchronous work — the API drops events
   on the **Service Bus** queue and returns immediately, so slow jobs never make the
   user wait.
5. A background consumer later reads the queue and sends **Email** (weekly digest +
   notifications) via Azure Communication Services — best-effort, so if email is
   down a book can still be listed.

---

## The containers (box = a running thing)

| Container | Technology | Its job |
|-----------|------------|---------|
| Mobile app | React Native | The member-facing UI |
| API service | Node.js Express on Azure App Service | The REST API + background queue consumer |
| Database | Azure SQL | System of record: books, members, loans |
| Cache | Azure Cache for Redis | Hot reads (a book, recent list, searches) |
| Object store | Azure Blob Storage | Book photos |
| Queue | Azure Service Bus | Buffers digest + notification work |
| Email | Azure Communication Services | Sends outbound email |
| Identity | Microsoft Entra External ID | Issues and signs JWTs |

---

## The two kinds of arrow (this is the key idea)

- **Solid black = synchronous** — the caller *waits* for an answer (e.g. the API
  waiting on a SQL query). Every arrow is labelled with its protocol
  (HTTPS / SQL / TCP) and a verb ("Read/write books").
- **Dashed orange = asynchronous** — fire-and-forget via the queue. The API drops a
  message and moves on; a consumer handles it later. This is why listing a book
  succeeds even if email is down.

**Colour key:** blue = app we build · teal = data/cache · orange = queue ·
grey = managed/external service.

---

## The four flows the diagram supports

- **List a book** → API writes to SQL, returns `201`, queues a "book added" event.
- **Search / view a book** → API checks Redis first, falls back to SQL, warms the cache.
- **Borrow a book** → API writes a borrow request to SQL, queues a notification.
- **Return a book** → `PATCH /loans/{id}` flips the loan to `returned` in SQL.

---

## What to say when explaining the diagram

- "The user talks to the app, the app talks to the API — never the user to the API."
- "Solid arrows wait; dashed orange arrows don't — that's sync vs async."
- "The cache sits beside the DB on the read path; the queue sits on the write path
  for slow/optional work."
- "The one thing that proves the design: a book listing can't be blocked by email,
  because email is on the async (dashed) side of the queue."
