# BookSwap Mock Tests — Bruno collection

**What it is:** a small [Bruno](https://usebruno.com) collection that runs the 5
smoke tests from the mock report against the Prism mock server. Each request has an
assertion on its expected status, so **pass = pass**.

---

## Before you run anything

The mock server must be running. From the repo's `openapi/` folder:

```bash
npx @stoplight/prism-cli mock bookswap-openapi.yaml --port 4010
```

Leave it running (look for `Prism is listening on http://127.0.0.1:4010`).

---

## Option A — run in the Bruno app (best for screenshots)

1. **Open Collection** → pick this `bookswap-mock-tests` folder.
2. Set the environment dropdown (**top-right**) to **Local**
   (this sets `baseUrl = http://127.0.0.1:4010`).
3. Click a request → **Send**, and read the status + Assertions on the right.
4. To run all at once: hover the collection → **⋯** / right-click → **Run**.

If you skip step 2, every request fails with a blank/URL error — the #1 gotcha.

## Option B — run from the command line

From inside this folder (the one containing `bruno.json`):

```bash
npx @usebruno/cli run --env Local
```

You'll get a summary table: **5 (5 Passed), Assertions 5/5**.

---

## What each request checks

| Request | Sends | Expects |
|---------|-------|---------|
| 01 List books | GET `/books?page=1&pageSize=20` + token | 200 |
| 02 Create book – valid | POST `/books` with full body | 201 |
| 03 Create book – missing title | POST `/books` without `title` | 422 |
| 04 Borrow request id=999 | POST `/books/999/borrow-requests` | **422** (the finding) |
| 05/06 List books – no auth | GET `/books` with no token | 401 |

**About request 04:** it asserts **422** on purpose. The id `999` isn't a UUID, so
the mock rejects the URL shape — that's the documented "Finding 1" in the report,
not a bug.
