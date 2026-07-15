# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI as an assistant throughout, and verified everything it produced against
the actual code, the test results, and the Conventional Commits spec before
committing. Specific uses:

- **Codebase orientation / understanding a pattern (Comments 1–3).** Before
  writing the deduplication, I had AI walk me through `add_to_collection()` in
  `collection_service.py` — specifically what its duplicate check does and that
  it *raises* `AlreadyInCollectionError` *before* `db.session.add`/`commit`
  (rather than returning). I confirmed that against the source, then wrote the
  parallel guard in `add_to_watchlist()` myself. I used the same approach to
  understand the fixture structure in `test_collection.py` before writing
  `test_watchlist.py`.

- **Stress-testing my design arguments (Comments 4 & 5).** After drafting my
  positions I asked AI what counterargument a careful reviewer would raise and
  which tradeoff I wasn't acknowledging. For Comment 4 it pushed the
  privacy-by-design / data-minimization angle (the most-protective default is the
  correct one) harder than my draft had — I revised the "Tradeoff acknowledged"
  paragraph to concede that directly and to condition my position on disclosure +
  the per-entry toggle, rather than just asserting the social benefit. For
  Comment 5 it raised that recency-of-*add* is a weak proxy for what you actually
  want to watch (watchlists are things you delay); I already leaned toward
  configurable sort, so I folded that objection in explicitly as the long-list
  failure mode and named user-selectable sort as the real fix. The arguments and
  final wording are my own.

- **Verifying commit format.** Before finalizing history I gave AI my
  `git log --oneline` and asked whether the messages follow Conventional Commits
  and whether any bundled more than one logical change. It flagged the
  rebase-resolution commit as the one combining several edits; I checked it
  against the spec myself and kept it as one deliberate atomic commit (every part
  serves the single goal of reconciling the feature with the UUID rebase), and I
  reclassified the rename from `refactor:` to `fix:` to fit the project's allowed
  type set (`feat`/`fix`/`test`/`docs`).

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` and updated its one call site — the import and
the call in `routes/watchlist/watchlist.py`. Committed on its own
(`fix: rename save_to_watchlist to add_to_watchlist per naming convention`).

**How I verified:** To find every call site I ran a project-wide search
(`grep -rn "save_to_watchlist" --include="*.py" .`) before editing — it returned
exactly three hits: the definition, the import, and the call. After renaming I
re-ran the same search and it returned nothing, confirming no reference was
missed. Full suite green (`pytest tests/ -v`, 5 passed).

## Comment 2 — Deduplication
**What I did:** Added deduplication to `add_to_watchlist()`, mirroring
`add_to_collection()`. Before inserting, it queries for an existing
`WatchlistEntry` with the same `user_id`/`film_id`; if one exists it raises a new
`AlreadyInWatchlistError` instead of creating a duplicate row. Committed
separately from the rename
(`fix: add deduplication check to prevent duplicate watchlist entries`).

**How I verified:** I read `add_to_collection()` to understand the pattern: it
does `CollectionEntry.query.filter_by(user_id=..., film_id=...).first()` and,
if a row is returned, raises `AlreadyInCollectionError` *before* reaching
`db.session.add`/`commit` — so on a duplicate it raises rather than returns. I
reproduced that exact shape with `WatchlistEntry` and `AlreadyInWatchlistError`.
`test_add_to_collection_duplicate_raises` documents the expected behavior
(second add raises; only one row persists). Full suite green.

_Follow-up (resolved):_ the deduplication above is enforced in the service, but
the watchlist **route** originally had no `try/except`, so a duplicate add
propagated uncaught and returned HTTP 500 instead of a 409. I later added the
`try/except` to the add endpoint so it now returns **409** for
`AlreadyInWatchlistError` and **404** for `FilmNotFoundError`, matching the
collection route (`fix: return 404/409 from watchlist add route instead of 500`).

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with
`test_add_to_watchlist_nonexistent_film_raises`, asserting that adding a
`film_id` with no matching row raises `FilmNotFoundError`.

**How I verified:** I modeled the test on
`test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` —
same `app`/`sample_user` fixtures (there is no shared `conftest.py`; fixtures are
defined per file) and the same `with pytest.raises(...)` structure. One
adaptation: I originally used a nonexistent integer id (`999999`) because
`Film.id` was still an integer on the feature branch. After the rebase onto
`main` (Comment 6) migrated film IDs to UUIDs, I updated the fake id to a UUID
string (`"00000000-..."`), matching the collection test. Ran
`pytest tests/test_watchlist.py -v` (1 passed) and the full suite
`pytest tests/ -v` (5 passed).

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for watchlist entries.

**Reasoning:** CineLog is a *social* film-logging platform, not a private
notes app, and the watchlist is one of its most social objects — "films I want
to watch" is exactly the signal that drives recommendations, watch-alongs, and
"oh, add this one too" conversations between friends. I'm optimizing for the
user who joins to be part of that: a public-by-default watchlist means their
queue is visible to friends from day one, so the discovery and social features
have real data to work with. The alternative — private by default — runs
straight into the empty-network problem: if every user has to opt in before
anything is shared, the social surface starts empty, the discovery feed shows
nothing, and the feature that motivated the platform never gets off the ground.
It's also consistent with CineLog's existing posture: `CollectionEntry` has no
visibility flag at all, so watched-films are already effectively open. Defaulting
the watchlist to public keeps the two features coherent rather than making the
watchlist the one mysteriously-private surface.

**Tradeoff acknowledged:** Public-by-default opts users into sharing before they
necessarily understand the visibility model, and a watchlist can be *more*
revealing than a watched-log — it signals intent and current headspace (someone
queuing films about a diagnosis, a breakup, or their sexuality is broadcasting
something they may not have chosen to). The privacy-by-design / data-minimization
argument (the most protective default is the correct one) is real, and I'm
accepting that cost rather than dismissing it. My position holds *only if* it's
paired with (a) clear disclosure at the point of adding a film that the entry is
public, and (b) the per-entry `public` toggle that already exists on the model,
surfaced in the UI so a user can make any single entry private without hunting
through settings. If CineLog's audience skewed toward minors or a
privacy-sensitive niche, I'd flip the default — the social thesis is what
justifies it, not a claim that public is universally safer.

## Comment 5 — Sort order
**My position:** Adopt the maintainer's preference — sort `get_watchlist()` by
`date_added` descending (newest first). Implemented in
`services/watchlist_service.py` (`feat: sort watchlist by date added, newest
first`).

**Reasoning:** A watchlist is a *queue of intent*, and the dominant task when a
user opens it is "what do I want to watch next?" The film they added most
recently is the freshest signal of current interest — a trailer they just saw, a
recommendation from this afternoon — so newest-first puts the most relevant
items where the eye lands. Alphabetical optimizes for a different task, *lookup
by title* ("where's Dune?"), which is rarer on a watchlist than on a full
catalog; and if you already know the exact title you're usually not scanning the
list to begin with. So recency serves the common case and alphabetical serves
the uncommon one.

**Engagement with reviewer's point:** I think the maintainer is right on both
counts and I'm not just deferring. On **consistency**: `get_collection()`
already sorts `date_added.desc()`, and two sibling features sorting differently
is a genuine UX and maintenance cost — anyone reading both services has to hold
two mental models, and the implementations now converge into near-identical
shape, which is easier to reason about and test. On **recency = relevance**: it
matches how the queue is actually used. Where I'll push on the maintainer's
framing is that recency-of-*add* is an imperfect proxy — the whole point of a
watchlist is that you *don't* watch things immediately, so a film added months
ago that you still want to see gets buried at the bottom, and for a user with a
long list that's a real failure mode. I don't think that overturns the decision:
for a *single* default, recency still beats alphabetical for the "what's next"
glance, and the correct fix for the long-list case isn't A–Z, it's a
user-selectable sort. I'd log that as a follow-up if watchlist length becomes a
pain point, but I wouldn't hold up adopting date-added now to wait for it.

## Comment 6 — Rebase
**What conflicted:** Nothing surfaced as a textual conflict — and that was the
trap. `git rebase origin/main` first refused to start ("untracked working tree
files would be overwritten by checkout: .gitignore") because my local
`.gitignore` was an empty untracked file while `origin/main` has a real,
tracked one. After removing the empty file the rebase completed and reported
success with zero conflict markers. But the rebase had silently dropped the
`WatchlistEntry` model, breaking the entire feature (`ImportError: cannot import
name 'WatchlistEntry'`).

Why it happened: `WatchlistEntry` was never added by any feature-branch commit —
it was inherited from the shared base commit (`014ae54`), where `models.py`
already defined it with an integer `film_id`. Main's refactor
(`07ca580`, "migrate film IDs from integer to UUID") rewrote `models.py`,
migrated `Film.id` to a UUID, and removed `WatchlistEntry` (main had no
knowledge of the watchlist work). Because the feature branch made *no* changes
to `models.py` relative to the base, the rebase had no watchlist-side edits to
replay and simply took main's version of the file — so `WatchlistEntry`
disappeared with no conflict to flag.

**How I resolved it:**
1. Removed the empty untracked `.gitignore` (it held nothing;
   `origin/main`'s tracked version is the correct one) so the rebase could run.
2. Restored `WatchlistEntry` in the post-refactor `models.py`, adapted to the
   new schema: `film_id` is now `String(36)` (UUID) to match `Film.id`, not the
   old `Integer` — a `film_id` of the wrong type would be a broken foreign key.
3. Added a `Film.watchlist_entries` relationship (`backref="film"`) so
   `get_watchlist()`'s `entry.film` access resolves. The base model never had
   this, so `get_watchlist` was latently broken; since it had no test, nothing
   caught it.
4. Updated the nonexistent-film test to use a UUID id instead of the old integer.
5. Swept the remaining watchlist code for lingering integer-ID references: the
   service docstring described `film_id (int)` and the route's example body
   showed `{ "film_id": <int> }`. Updated both to the UUID form (`str` /
   `"<uuid>"`), matching the collection service.

Steps 2–5 are all part of the rebase resolution and, after I cleaned up the
branch history, live in a single commit:
`fix: rebase onto main — restore WatchlistEntry as UUID`.

**How I verified no conflict remains:**
- `git status` is clean.
- `git log --merges origin/main..HEAD` returns nothing, and
  `git log --oneline origin/main..HEAD` shows a linear history of my commits on
  top of `origin/main` — **no merge commits** remain, confirming a true rebase
  rather than a merge.
- `grep -niE "\bint\b|integer|<int>|pre-refactor"` over the watchlist service and
  route returns nothing — no integer-ID references left.
- `pytest tests/ -v` passes all 5 tests (previously the watchlist test module
  failed to even import). I also ran a live end-to-end smoke test in an app
  context: adding two films then calling `get_watchlist()` returns them
  newest-first with `public=True`, a duplicate add raises
  `AlreadyInWatchlistError`, and an unknown `film_id` raises `FilmNotFoundError`
  — confirming the model, relationship, dedup, sort, and UUID-typed foreign key
  all work together after the rebase.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog: a per-user list of films a user wants to watch
(distinct from the collection, which is films they've already watched). It
introduces:

- `WatchlistEntry` model — links a user to a film, with a `date_added` timestamp
  and a `public` visibility flag. `film_id` is a UUID (`String(36)`) matching
  the post-refactor `Film.id`.
- `services/watchlist_service.py` — `add_to_watchlist(user_id, film_id)` and
  `get_watchlist(user_id)`. Adding validates the film exists
  (`FilmNotFoundError`) and rejects duplicates (`AlreadyInWatchlistError`).
- `routes/watchlist/watchlist.py` — `GET /watchlist/<user_id>` and
  `POST /watchlist/<user_id>/add`, registered under the `/watchlist` prefix.

### Design decisions
1. **Default visibility — watchlist entries are public by default
   (`public=True`).** CineLog is a social film platform and the watchlist is one
   of its most social objects; a public default seeds the discovery/social
   surface instead of leaving it empty (the opt-in "empty network" problem), and
   is consistent with `CollectionEntry`, which has no visibility flag at all. The
   accepted tradeoff is privacy: a public default opts users in before they may
   understand it, so the position is conditioned on clear disclosure at add-time
   plus the per-entry `public` toggle. (Full reasoning under **Comment 4**.)
2. **Sort order — `get_watchlist()` returns entries newest-first by
   `date_added`.** This adopts the maintainer's preference over the original
   alphabetical sort: it matches `get_collection()` (consistency across the two
   features) and surfaces the most recently added film first, which best serves
   the "what do I want to watch next?" glance. The tradeoff — a long-standing
   item sinks to the bottom — is best solved later by a user-selectable sort.
   (Full reasoning under **Comment 5**.)

### How to manually test the feature
The catalog and users are seeded (there are no create endpoints for them), so
first seed one user and two films, then exercise the endpoints.

1. **Install and prepare:**
   ```bash
   pip install -r requirements.txt
   ```
2. **Seed a user and two films** (prints the IDs you'll use below):
   ```bash
   python - <<'PY'
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       u = User(username="alice", email="alice@example.com")
       arrival = Film(title="Arrival", year=2016, genre="Sci-Fi")
       dune = Film(title="Dune", year=2021, genre="Sci-Fi")
       db.session.add_all([u, arrival, dune]); db.session.commit()
       print("USER_ID   =", u.id)
       print("ARRIVAL   =", arrival.id)
       print("DUNE      =", dune.id)
   PY
   ```
3. **Start the app** (http://localhost:5000):
   ```bash
   python app.py
   ```
4. **Add Arrival, then Dune** to alice's watchlist (use the IDs from step 2):
   ```bash
   curl -X POST http://localhost:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<ARRIVAL>"}'
   curl -X POST http://localhost:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<DUNE>"}'
   ```
   Each returns `201` with the new entry JSON, including `"public": true`
   (visibility decision).
5. **View the watchlist** and confirm sort order:
   ```bash
   curl http://localhost:5000/watchlist/<USER_ID>
   ```
   Expect both films **newest-first** — `Dune` (added last) before `Arrival`
   (sort decision) — each with `"public": true`.
6. **Verify deduplication and error handling:** re-run the Arrival add from
   step 4 — it returns **`409`** (`AlreadyInWatchlistError`), and re-fetching the
   watchlist shows the film still appears **only once**. Adding an unknown
   `film_id` returns **`404`** (`FilmNotFoundError`), and a request with no
   `film_id` returns **`400`** — matching the collection route.
7. **(Optional) Run the automated tests:**
   ```bash
   pytest tests/ -v
   ```
