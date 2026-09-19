# Cloud sync — setup

Written 2026-09-19. Fifteen minutes, once. After this the ledger can carry the
same data between your machines.

Sync is **off until you finish this**, and off afterwards too unless you tick
auto-push. Until then the app behaves exactly as it always has: one browser, one
`localStorage` key, and no network request of any kind — verified, not assumed:
with nothing configured the page makes zero requests.

**Sync does not work from a file on disk.** A page opened over `file://` has no
origin, and no database will accept a call from one. Use
https://talon270.github.io/ledger/ for syncing; the disk copy stays local-only,
which is exactly what it is for.

---

## 1 · Create the database

[turso.tech](https://turso.tech) → sign up → create a database. The free tier is
far more than this needs: one row, a few hundred kilobytes.

```sh
# or from the CLI
turso db create ledger
turso db show ledger --url          # → libsql://ledger-<you>.turso.io
turso db tokens create ledger       # → eyJhbGciOi…
```

Pick the region nearest you. Mumbai (`bom`) makes a sync a few tens of
milliseconds instead of a few hundred.

## 2 · Paste both into the app

**data → ☁ cloud sync**. Put the URL in as Turso gives it to you — `libsql://…`
is converted to `https://…/v2/pipeline` for you — then the token, then a device
label so you can tell which machine wrote the copy on the server.

Press **test & create table**. That runs one statement:

```sql
CREATE TABLE IF NOT EXISTS ledger_state(
  id         TEXT PRIMARY KEY,
  data       TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  device     TEXT
);
```

There is no SQL editor step and nothing else to create. The whole ledger lives
in one row, `id = 'main'`, as the same JSON the backup export produces.

## 3 · Push once, then pull on the other machine

**push ↑** writes this browser's ledger to that row. On the second machine, open
the same page, paste the same URL and token, and press **pull ↓**.

---

## What it will and will not do to your data

| | |
|---|---|
| **A pull shows both sides before it replaces anything** | Counts of expenses, receipts, categories, goals, holdings, contributions and closed months, here versus the server, plus who wrote the server copy and when. Cancel is the keyboard default |
| **A pull is undoable** | It takes a snapshot first, so `u` brings this browser's copy back |
| **A corrupt row cannot destroy your ledger** | The incoming JSON goes through the same validator as an imported backup. If it does not parse as a ledger, the pull aborts and nothing local is touched |
| **A push that would clobber a newer copy asks first** | If the server row changed after your last sync, you get the other machine's name and timestamp, and have to confirm |
| **Auto-push never opens a dialog** | If the server moved on it refuses the automatic push and says so in the status line, rather than ambushing you with a modal four seconds after a keystroke |
| **A failed sync costs you nothing** | The local save always happens first and never waits for the network. Every failure path leaves local data untouched and prints why |

**What it is not: a merge.** The row holds the whole ledger, so a push replaces
the server copy and a pull replaces this browser's. If you log expenses on two
machines without syncing between them, one set of entries will lose — the
dialog tells you which, but it cannot combine them. Sync on the machine you
just used, before you move to the other one.

---

## The token

It is stored in this browser in plain text, under its own storage key, and
anything with access to this browser can read it. Two consequences worth
acting on:

- **Scope it.** `turso db tokens create ledger` is already scoped to that one
  database. Do not paste an account-wide token.
- **It never lands in a backup.** Credentials live under `ledger.sync.v1`, not
  inside the ledger itself, so the JSON export you might mail to yourself
  carries no token. This is deliberate — verified by checking that the exported
  state contains no part of it.

Rotate with `turso db tokens invalidate ledger` if a machine is lost or shared.

---

## If the browser blocks it

**This is the one thing I could not confirm in advance.** Turso documents its
HTTP API for servers and edge runtimes, where cross-origin rules do not apply,
and says nothing either way about being called from a browser page on another
origin. If it does not send `Access-Control-Allow-Origin`, your browser will
refuse the response before the app ever sees it — and the app will tell you so
in those words, because a `fetch` that fails that way comes back with no HTTP
status at all.

Test it in ten seconds from a terminal, where CORS does not apply:

```sh
curl -sS https://ledger-<you>.turso.io/v2/pipeline \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"requests":[{"type":"execute","stmt":{"sql":"select 1"}},{"type":"close"}]}'
```

If that returns JSON but the app reports a blocked request, it is CORS, and this
worker fixes it — deploy it once on Cloudflare (free), then paste the *worker's*
URL into the app instead of the Turso one:

```js
// ledger-sync-proxy — adds CORS to Turso's HTTP API. Nothing else.
const ALLOW = "https://talon270.github.io";   // only your page, not the world
const DB    = "https://ledger-<you>.turso.io/v2/pipeline";

export default {
  async fetch(req) {
    const cors = {
      "Access-Control-Allow-Origin": ALLOW,
      "Access-Control-Allow-Headers": "authorization,content-type",
      "Access-Control-Allow-Methods": "POST,OPTIONS",
      "Access-Control-Max-Age": "86400",
    };
    if (req.method === "OPTIONS") return new Response(null, { headers: cors });
    const upstream = await fetch(DB, {
      method: "POST",
      headers: {
        authorization: req.headers.get("authorization") ?? "",
        "content-type": "application/json",
      },
      body: await req.text(),
    });
    return new Response(upstream.body, { status: upstream.status, headers: { ...cors, "content-type": "application/json" } });
  },
};
```

The proxy holds no credentials: it forwards the `Authorization` header your
browser sends and nothing else, so the token still only exists in your browser
and in Turso. Set `ALLOW` to your page's origin rather than `*`, or anyone's
page can make your browser's requests for them.

The app appends `/v2/pipeline` to whatever you paste unless it is already there,
so give it the worker URL with the path included.
