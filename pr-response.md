# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI as an assistant and verified everything against the code, the tests,
and the Conventional Commits spec before committing. Three specific uses:

- **Understanding the codebase (Comments 1–3):** I had AI walk me through the
  existing `add_to_collection()` and the collection tests so I understood the
  duplicate-check pattern and the test structure, then wrote the watchlist
  versions myself.
- **Stress-testing my arguments (Comments 4–5):** After drafting my positions I
  asked AI for the strongest counterargument. For Comment 4 it pushed the
  privacy-first angle, so I added that tradeoff and conditioned my position on
  disclosure plus a per-entry toggle. For Comment 5 it noted that "recently
  added" is a weak proxy for what you'll actually watch, so I folded that in and
  pointed to a user-selectable sort as the real fix. The arguments are my own.
- **Checking commit format:** I asked AI to review my commit log for
  conventional-commit compliance and bundled commits, verified against the spec
  myself, and reclassified the rename from `refactor:` to `fix:`.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` and updated
its one caller in the watchlist route.

**How I verified:** I searched the whole project for the old name before editing
and found three references — the definition, the import, and the call. After
renaming I searched again and found none, confirming nothing was missed. All
tests still pass.

## Comment 2 — Deduplication
**What I did:** `add_to_watchlist()` now checks whether the user already has that
film on their watchlist and, if so, raises `AlreadyInWatchlistError` instead of
adding a duplicate.

**How I verified:** I followed the existing `add_to_collection()` pattern — it
looks up an existing entry and raises *before* saving, so a duplicate is rejected
rather than written. I mirrored that exactly for the watchlist.

**Follow-up:** The route originally let that error surface as a 500. I later
wrapped the endpoint so a duplicate returns **409** and an unknown film returns
**404**, matching the collection route.

## Comment 3 — Missing test
**What I did:** Added `tests/test_watchlist.py` with a test asserting that adding
a film that doesn't exist raises `FilmNotFoundError`.

**How I verified:** I modeled it on the collection test for the same case,
reusing the same fixtures and structure, and used a nonexistent UUID as the film
id (matching the collection test after the UUID migration). The test passes, and
so does the full suite.

## Comment 4 — Default visibility
**Position:** Keep watchlist entries **public by default**.

**Reasoning:** CineLog is a social platform, and a watchlist is one of its most
social signals — what you plan to watch drives recommendations and shared
viewing. A public default means friends see your queue from day one, so
discovery has something to work with; a private default hits the empty-network
problem, where nothing is shared and the social features never take off. It's
also consistent with the collection, which has no privacy flag at all.

**Tradeoff:** A public default opts people in before they fully understand it,
and a watchlist can reveal intent more than a watched-list. The privacy-first
argument (the most-protective default is the right one) is legitimate, so my
position depends on clear disclosure when adding a film plus the per-entry toggle
the model already has. For a privacy-sensitive audience or minors, I'd flip the
default.

## Comment 5 — Sort order
**Position:** Adopt the maintainer's preference — **newest-first by date added**.

**Reasoning:** A watchlist is a queue of intent, and the main question when you
open it is "what do I watch next?" The most recently added film is the freshest
signal of interest, so newest-first puts it where you look. Alphabetical only
helps when hunting a specific title, which is rarer here.

**Engaging the maintainer:** I agree on both their points — it's consistent with
the collection (which already sorts newest-first) and recency usually matches
intent. My one pushback is that "recently added" is an imperfect proxy, since
watchlists are things you deliberately delay, so an old-but-wanted film sinks to
the bottom. That doesn't change the default — recency still beats alphabetical
for the common case — but the real fix for long lists is a user-selectable sort,
which I'd add as a follow-up.

## Comment 6 — Rebase
**What conflicted:** The rebase reported success with no conflict markers, but it
silently dropped the `WatchlistEntry` model and broke the feature. The model had
been inherited from the original base commit, and main's "integer → UUID"
refactor rewrote the models file without it. Because my branch never modified
that file, the rebase just took main's version and the model vanished. (It also
initially refused to start because of an empty, untracked `.gitignore`, which I
removed.)

**How I resolved it:** I restored the `WatchlistEntry` model against the new
schema — using a UUID film id to match `Film`, and adding the film relationship
that `get_watchlist()` needs — then updated the last few places in the watchlist
code that still referred to integer ids. This is the core integer-vs-UUID
conflict the maintainer flagged.

**How I verified:** Tests pass (they previously couldn't even import), the history
is linear with no merge commits, and a manual run confirms that adding films,
deduplication, newest-first ordering, and the UUID foreign key all work together.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog — a per-user list of films someone wants to
watch, separate from their collection of films already watched. Users can add a
film to their watchlist and view the list back, sorted newest-first. Adding a
film that doesn't exist, or one already on the list, is rejected with a clear
error.

### Design decisions
1. **Visibility — public by default.** Watchlists are a social signal, so they're
   shared by default to power discovery; the tradeoff is privacy, handled by
   disclosure and a per-entry toggle. (See Comment 4.)
2. **Sort order — newest-first by date added.** Matches the collection and
   surfaces the film you most recently wanted to watch. (See Comment 5.)

### How to manually test the feature
1. Install dependencies and start the app (`python app.py`, runs on
   `localhost:5000`).
2. The catalog and users are seeded (there are no create endpoints), so create
   one user and two films in a Python shell and note their ids.
3. `POST` each film id to `/watchlist/<user_id>/add` — each returns **201** with
   `"public": true`.
4. `GET /watchlist/<user_id>` — both films come back **newest-first** (the one
   added last appears first).
5. Re-add the first film — it returns **409** and the list still shows it only
   once. Adding an unknown film id returns **404**, and a request with no film id
   returns **400**.
6. Optionally run `pytest tests/` to run the automated tests.
