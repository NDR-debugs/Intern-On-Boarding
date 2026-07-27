# BookSwap — Backend Design (Day 2)

A neighbour-to-neighbour book lending marketplace, one apartment building at a time.
This repo is a **design only** — no running code — so another developer could pick
it up on Day 3 and start building.

---

## What's in here

```
bookswap-design/
├── openapi/
│   ├── bookswap-openapi.yaml     # the REST API contract (validated)
│   └── README.md                 # how to read + validate the spec
├── decisions/
│   ├── storage-decisions.md      # where each piece of data lives + cache/queue plan
│   ├── mock-report.md            # results of running the spec as a mock (with screenshots)
│   └── screenshots/              # Bruno test screenshots used by the report
├── diagrams/
│   ├── container-diagram.png     # the C4 Level 2 picture
│   ├── container-diagram.drawio  # editable source for the diagram
│   └── README.md                 # how to read the diagram
├── bruno/
│   └── bookswap-mock-tests/      # Bruno collection that runs the 5 smoke tests
└── README.md                     # you are here
```

**Each folder has its own README** — start there for detail on that piece.

---

## The four deliverables

| # | Deliverable | File |
|---|-------------|------|
| 1 | OpenAPI spec | `openapi/bookswap-openapi.yaml` (+ `openapi/README.md`) |
| 2 | Storage & cache decisions | `decisions/storage-decisions.md` |
| 3 | Container diagram | `diagrams/container-diagram.png` (+ `diagrams/README.md`) |
| 4 | Mock smoke-test report | `decisions/mock-report.md` |

---

## The one-minute story (how the pieces fit)

- **List a book** → API writes to SQL, returns `201`, and queues a "book added"
  event. Email being down can't block the listing.
- **Search / view a book** → API tries Redis first (fast), falls back to SQL, warms
  the cache. Search stays under 300 ms for 5,000 books.
- **Borrow a book** → API creates a borrow request in SQL and queues a notification
  so the owner hears about it in-app within ~2 seconds.
- **Return a book** → `PATCH /loans/{id}` flips the loan to `returned` in SQL.

**The rule tying it together:** SQL + Blob are the *source of truth*; Redis and the
queue are disposable *helpers*.

---

## Quick start (validate + mock the API)

```bash
cd openapi
npx @apidevtools/swagger-cli validate bookswap-openapi.yaml   # prints "... is valid"
npx @stoplight/prism-cli mock bookswap-openapi.yaml --port 4010
```

Then run the 5 tests — see `bruno/bookswap-mock-tests/README.md` and
`decisions/mock-report.md`.
