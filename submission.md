# Mixtape — Codebase Map

Mixtape is a small Flask + SQLAlchemy backend for a music-sharing app. Users share
songs, add them to (collaborative) playlists, rate them, "listen" to them (which
builds a daily streak), and see a feed of what their friends are playing.

---

## How I used AI on this project

I used an AI assistant (Claude) as a pair-programming partner throughout this
project, but I drove the understanding of each bug myself before we changed any
code. My general loop was: ask the AI to explain a part of the system, form my own
hypothesis about what was wrong, then confirm it against the actual code and a real
run before fixing.

**Understanding the codebase first.** Before touching any bugs, I asked the AI to
help me build a mental model of how the app is wired together. It produced the
layered diagram and the routes-to-services mapping in this doc. Once I understood that routes just delegate to services and all
the real logic lives in `services/`, I knew that every bug would live in a service
function, which is exactly where they turned out to be.

**Working each bug together.** For each of the five bugs I had the AI trace the path
from the endpoint down to the responsible service function, then I read the flagged
function myself to confirm the specific line before we fixed it. Some concrete
examples of what I asked it to explain or trace:
- *Streak (Bug 1):* it traced `POST /songs/<id>/listen` → `record_listening_event()`
  → `update_listening_streak()` and pointed at the `today.weekday() != 6` clause. I
  confirmed for myself that `weekday()` returns 6 for Sunday, which is what made the
  behaviour "reset every Sunday" rather than randomly.
- *Notification asymmetry (Bug 4):* I asked it to compare `rate_song()` against
  `add_to_playlist()` side by side, which made the missing `create_notification()`
  call obvious. We then mirrored the existing playlist pattern for the fix.
- *Playlist (Bug 5):* the two failing tests already pointed here; the AI confirmed the
  `songs[:-1]` slice was dropping the last element and I checked that the empty-list
  case (`[][:-1]`) explained why one of the three tests still passed.

**Where I had to verify things myself / the AI was incomplete or wrong.** This is the
part I learned the most from:
- *The search bug (Bug 3) did not reproduce the way the AI predicted.* The AI
  confidently said the `outerjoin(song_tags)` would return the same song multiple
  times, but when I ran the search tests they all **passed**. Rather than trust the
  explanation, I ran the query directly and found that the raw SQL join really does
  produce 3 rows for a 3-tag song, but **SQLAlchemy 2.0.50 deduplicates single-entity
  results by primary key**, so the duplication never reached the caller in this
  environment. The AI's root cause (pointless join) was right, but its claim that the
  bug was *currently visible* was wrong — the bug was latent and version-dependent. I
  only learned that by checking instead of taking the explanation at face value.
- *The AI's initial "all five bugs reproduce" summary was partly stale.* When we
  started on the streak bug, the AI expected the Sunday test to fail, but it passed —
  because that fix had already been committed earlier. I had to check `git log`/`git
  diff` to see the true state of the tree rather than rely on the AI's assumption.
- *A number in the AI-written notes didn't match the committed code.* The draft RCA
  said the feed window was changed to 15 minutes, but the value actually committed was
  10 minutes. I caught this by diffing the doc against the real source and corrected
  it. Takeaway: AI-generated prose can drift from the actual code, so every concrete
  value and claim in this doc was checked against a run or the committed source.

Every fix in this submission was confirmed with the test suite (`pytest`) or a
targeted reproduction script that I ran and read the output of — I did not accept a
fix as "done" on the AI's word alone.

---

The code is organized in **four layers**, and every request flows through them
top-to-bottom:

```
HTTP request
   │
   ▼
routes/      ← Blueprints. Parse input, call one service, format the JSON response.
   │
   ▼
services/    ← All business logic and every database query lives here.
   │
   ▼
models.py    ← SQLAlchemy table definitions.
   │
   ▼
app.py       ← Owns the `db` object and the app factory that wires it all together.
```

---

## Main files and what each one does

### `app.py` — the application factory + database handle
- Creates the single shared `db = SQLAlchemy()` object that **every other module
  imports** (`from app import db`).
- `create_app(config=None)` builds the Flask app, sets the SQLite config
  (`DATABASE_URL` env var, default `mixtape.db`), registers the four blueprints
  under URL prefixes, and calls `db.create_all()`.
- URL prefixes: `songs_bp → /songs`, `playlists_bp → /playlists`,
  `users_bp → /users`, `feed_bp → /feed`.

### `models.py` — the data model
Defines **7 entity models** and **3 association (join) tables**. All primary keys
are string UUIDs generated by `generate_uuid()`.

Entities:
- **`User`** — the hub. Has `listening_streak` and `last_listened_at` stored
  directly on the row (the streak is *not* recomputed from events; it's a running
  counter). Backref relationships to `Song`, `Rating`, `ListeningEvent`,
  `Notification`, and `Playlist`.
- **`Song`** — `title`, `artist`, `album`, `genre`, and `shared_by` (FK → the
  `User` who originally shared it). `shared_by` is what the notification system
  keys off of.
- **`Tag`** — just `id` + `name`.
- **`ListeningEvent`** — one row per "listen" (`user_id`, `song_id`, `listened_at`).
- **`Rating`** — `user_id`, `song_id`, `score` (1–5), with a
  `UniqueConstraint(user_id, song_id)` → **one rating per user per song**.
- **`Playlist`** — `name`, `created_by` (FK → User), `is_collaborative` (default
  `True`).
- **`Notification`** — `user_id` (recipient), `notification_type`, `body`, `read`.

Association tables:
- **`friendships`** — `User` ↔ `User`, symmetric self-join (used by the feed).
- **`song_tags`** — `Song` ↔ `Tag`, plain many-to-many.
- **`playlist_entries`** — `Playlist` ↔ `Song`, but a **rich join table**: it also
  stores `position` (explicit song ordering), `added_by`, and `added_at`. Songs in
  a playlist have an explicit order, not just insertion order.

### `routes/` — HTTP blueprints (thin controllers)
Each file defines one Blueprint. Routes parse request data, call **one** service
function, and turn the result (or a `ValueError`) into JSON. They contain no
business logic.

| File | Endpoints | Delegates to |
|------|-----------|--------------|
| `routes/songs.py` | `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen` | `search_service`, `notification_service`, `streak_service` |
| `routes/playlists.py` | `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs` | `playlist_service`, `notification_service` |
| `routes/users.py` | `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read` | `streak_service`, `notification_service` (+ `models` directly) |
| `routes/feed.py` | `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity` | `feed_service` |

### `services/` — business logic + database access
- **`search_service.py`** — `search_songs(query)` (case-insensitive `ilike` match
  on title *or* artist, joined with tags) and `get_song(id)`.
- **`playlist_service.py`** — create a playlist, list a user's playlists, and
  `get_playlist_songs(id)` which joins through `playlist_entries` and orders by
  `position`.
- **`streak_service.py`** — `record_listening_event()` writes a `ListeningEvent`
  and then calls `update_listening_streak()`, which compares today's date to
  `user.last_listened_at` and either increments, holds, or resets the streak.
- **`notification_service.py`** — creates and reads `Notification`s. Also owns
  `add_to_playlist()` (adds a song to a playlist **and** notifies the sharer) and
  `rate_song()`.
- **`feed_service.py`** — `get_friends_listening_now()` (friends' listens within a
  24h window, deduped to one song per friend) and `get_activity_feed()` (most
  recent N friend listens, no time filter).

### `seed_data.py` — dev/test fixture loader
Uses `create_app()` + the models to populate the database with sample users,
songs, tags, playlists, listens, etc. **This is the only place `Song`s are
actually created** — there is no "share a song" HTTP endpoint.

### `tests/` — pytest suites
`test_playlists.py`, `test_search.py`, `test_streaks.py`. Each spins up an app via
`create_app()` and calls service functions directly.

---

## Data flow: adding a song to a playlist → notification

This is the real "an action notifies the original sharer" flow in the app (the
notification is triggered by a *playlist add*, not by a rating).

1. **Request** — `POST /playlists/<playlist_id>/songs` with JSON
   `{ "song_id": ..., "added_by": ... }`.
2. **Route** — `add_song()` in `routes/playlists.py` validates that both fields are
   present, then calls `add_to_playlist(playlist_id, song_id, added_by)`.
3. **Service** — `add_to_playlist()` in `services/notification_service.py`:
   - Loads the `Song`, the adding `User`, and the `Playlist` (raising `ValueError`
     → 400 if any is missing).
   - Appends the song to `playlist.songs` (writes a `playlist_entries` row) and
     commits.
   - **Then checks `song.shared_by != added_by_user_id`** — i.e. only notify if the
     person adding the song is *not* the one who originally shared it.
   - If so, calls `create_notification(user_id=song.shared_by,
     type="song_added_to_playlist", body="<adder> added your song '<title>' to the
     playlist '<name>'.")`, which inserts a `Notification` row for the sharer.
4. **Later** — the sharer sees it via `GET /users/<id>/notifications`
   (→ `get_notifications()`), and can `POST /users/notifications/<id>/read`
   (→ `mark_as_read()`).

So the **`Song.shared_by` foreign key is the link** that turns "someone touched my
song" into a notification addressed to me.

*(A second interesting flow: `POST /songs/<id>/listen` → `record_listening_event()`
writes a `ListeningEvent` and mutates `User.listening_streak` in the same call —
one endpoint with a side effect on a different table.)*

---

## Patterns I noticed

1. **Strict layering — routes never touch the database directly** (with one
   exception: `routes/users.py` does a `db.session.get(User, ...)` inline). Almost
   every route is 3–5 lines: parse JSON → call a service → `jsonify`. All logic and
   queries live in `services/`.

2. **One shared `db`, imported everywhere.** `app.py` defines it; `models.py` and
   every service do `from app import db`. `models.py` importing from `app` is the
   one "upward" dependency in an otherwise clean top-down graph.

3. **Errors travel as `ValueError`.** Services raise `ValueError("... not found")`;
   routes catch it and map it to a 404/400. There's no custom exception layer.

4. **Streak is denormalized state, not derived.** `User.listening_streak` /
   `last_listened_at` are counters updated on write, rather than being computed from
   `ListeningEvent` rows on read. Faster reads, but the counter can drift from the
   event history.

5. **`notification_service` is the busiest service** — used by three routes
   (songs, playlists, users). It also does a **function-local import** of
   `playlist_service` inside `add_to_playlist()` to avoid a circular import at
   module load time.

6. **`to_dict()` on every model** is the serialization boundary — services return
   plain dicts/lists, so routes can `jsonify()` without knowing about ORM objects.

---

<<<<<<< Updated upstream
# Root Cause Analysis

## Bug 1 — "My listening streak keeps resetting" (`streak_service.py`)

**How I reproduced it.**
The symptom is date-dependent, so I reproduced it with a fixed-date scenario rather
than waiting for a real calendar day. The reproducing sequence: a user listens on a
Saturday (streak becomes 1), then listens again the very next day, Sunday. Expected
result is a streak of 2 (two consecutive days); the actual result was 1 — the streak
reset even though no day was skipped. This is exactly what the existing test
`test_streak_increments_on_sunday` in `tests/test_streaks.py` pins down: it feeds
`2024-06-15` (Saturday) then `2024-06-16` (Sunday) into `update_listening_streak()`
and asserts the streak is 2. Against the original code that assertion fails; the bug
only surfaced when the *second* listen landed on a Sunday, which is why it looked
intermittent ("keeps resetting" ≈ resets every Sunday).

**How I found the root cause.**
I started from the endpoint the streak lives behind — `POST /songs/<id>/listen` in
`routes/songs.py` → `record_listening_event()` → `update_listening_streak()` in
`services/streak_service.py`. All the streak arithmetic is in that one function, so I
read its branch logic. The three branches key off `days_since_last` (0 = same day, 1
= consecutive, else = gap). The moment of confidence was reading the consecutive-day
branch and seeing a second, unrelated condition bolted onto it:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

The `days_since_last == 1` part is the correct "consecutive day" test. The
`and today.weekday() != 6` had nothing to do with consecutiveness — and 6 is exactly
the value that made the Sunday test fail. That pinned it to the specific line, not
just "somewhere in the streak logic."

**The root cause.**
Python's `date.weekday()` returns `6` for Sunday. The consecutive-day branch required
`today.weekday() != 6`, so whenever the current listen fell on a Sunday, the branch
condition evaluated to `True and False → False`, control fell through to the `else`,
and the streak was reset to 1 — *even though the user had listened the day before and
the streak should have incremented*. The gap-detection logic (`days_since_last`) was
correct on its own; the extra weekday clause was an unrelated condition that corrupted
a valid consecutive-day case one day out of every seven.

**My fix and side-effect check.**
I removed the `and today.weekday() != 6` clause so the branch reads
`elif days_since_last == 1:` — the streak now increments purely on the basis of
consecutive calendar days, which is the documented rule. I re-ran the full streak
suite (`tests/test_streaks.py`) to confirm the neighbouring behaviours still hold:
new-user start-at-1, increment on a normal consecutive day, no double-count on two
listens the same day, and reset after a genuinely skipped day all still pass (5/5).
Because the removed clause only ever *forced a reset*, deleting it cannot cause a
missed reset — the real reset path (`else`, for `days_since_last > 1`) is untouched.

## Bug 2 — "Friends Listening Now shows people from yesterday" (`feed_service.py`)

**How I reproduced it.**
There's no test for the feed, so I wrote a small reproduction script against an
in-memory database. It creates a user and a friend, makes them friends, and inserts a
single `ListeningEvent` for the friend timestamped **20 hours ago** (i.e. yesterday,
but within the last day). Then it calls `get_friends_listening_now(me.id)`. Expected
behaviour for a "listening *now*" feed is an empty result — the friend isn't
currently listening, they listened yesterday. Actual result: the feed returned that
friend, confirming stale listens leak into "now."

```
[A] only yesterday's listen -> feed size = 1   (BUG)
```

**How I found the root cause.**
Navigation path: `GET /feed/<id>/listening-now` in `routes/feed.py` →
`get_friends_listening_now()` in `services/feed_service.py`. Reading that function,
the query itself is correct — it filters `ListeningEvent.listened_at >= cutoff` where
`cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`, which correctly keeps events
*newer* than the cutoff. So the direction of the comparison was fine; the only other
input to the cutoff is the window size. Looking up the module-level constant made it
obvious:

```python
RECENT_THRESHOLD = timedelta(hours=24)
```

That was the moment of confidence: a 24-hour window is by definition "anything since
this time yesterday," which is exactly the reported symptom, and it's the single value
that defines what "now" means for this feed.

**The root cause.**
The "recent" window was set to 24 hours. `get_friends_listening_now()` treats any
friend with a listen in the last 24 hours as "listening now," so a listen from
yesterday afternoon still qualifies well into today. The feed was never meant to be a
day-long history (that's what `get_activity_feed()` is for, further down the same
file); it's supposed to show who is *currently* playing something. The window size,
not the query logic, was wrong.

**My fix and side-effect check.**
I changed the constant to `RECENT_THRESHOLD = timedelta(minutes=10)`, a window that
actually corresponds to "right now." I re-ran the reproduction script to check both
directions: a 20-hour-old (yesterday) listen is now excluded (feed size 0), and a
genuinely recent 5-minute-old listen still appears (feed size 1) — so the fix isn't
over-corrected. I also ran the full test suite; the feed change introduced no new
failures. `get_activity_feed()` is untouched — it deliberately does not use
`RECENT_THRESHOLD`, so the "full history" feed still behaves as before.

## Bug 5 — "The last song in a playlist never shows up" (`playlist_service.py`)

**How I reproduced it.**
Two existing tests in `tests/test_playlists.py` pin this down, so I ran them first to
confirm red. `seed_playlist` builds a playlist with 5 songs (Track 1–5) at explicit
positions 1–5. `test_playlist_returns_all_songs` asserts `get_playlist_songs()`
returns 5 songs; `test_playlist_returns_songs_in_order` asserts the titles are
`["Track 1", ..., "Track 5"]`. Both failed against the original code — the call
returned only 4 songs (Track 1–4), dropping Track 5:

```
2 failed, 1 passed
E  Right contains one more item: 'Track 5'
```

The third test (`test_empty_playlist_returns_empty_list`) passed even with the bug,
which was a useful clue about *where* the truncation was.

**How I found the root cause.**
Navigation path: `GET /playlists/<id>/songs` in `routes/playlists.py` →
`get_playlist_songs()` in `services/playlist_service.py`. The function's query is
correct — it joins through `playlist_entries` and orders by `position` ascending, so
`songs` is the full, correctly-ordered list. The bug had to be after the query, in
how the result was returned. The return line was:

```python
return [song.to_dict() for song in songs[:-1]]
```

The `[:-1]` slice was the moment of confidence: it's a list slice that excludes the
last element. That also explains why the empty-playlist test still passed — `[][:-1]`
is `[]`, so the truncation is invisible when there are no songs, and only shows up
once the playlist has at least one entry (where it silently drops the final one).

**The root cause.**
The list comprehension iterated over `songs[:-1]` instead of `songs`. `[:-1]` returns
every element *except the last*, so the highest-position song in every non-empty
playlist was discarded before serialization. The database query returned all songs
correctly; the truncation happened purely in the return statement. The function's own
docstring ("returns all songs in the playlist") contradicted the slice.

**My fix and side-effect check.**
I changed `songs[:-1]` to `songs` so the comprehension serializes the complete
ordered list:

```python
return [song.to_dict() for song in songs]
```

Both previously-failing playlist tests now pass (5 songs returned, correct Track 1–5
order), and the empty-playlist test still passes (an empty list is unaffected). I ran
the **full** suite afterwards: **13 passed**, 0 failed — this was the last outstanding
failure, so all three bug areas (streak, feed, playlist) are now green together with
no regressions.

## Bug 3 — "The same song keeps showing up twice in search" (`search_service.py`)

**How I reproduced it — and an important caveat.**
`tests/test_search.py` is built to expose this: `seed_songs` creates a song with
three tags ("Crown Heights Anthem") and the test
`test_search_no_duplicates_multi_tag_song` searches for it and asserts it appears
exactly once, with the comment "bug causes it to be 3." I ran the search suite first
to confirm — but **all five tests passed against the original code**. So I could not
reproduce the *visible* duplicate in this environment, and I stopped to find out why
before assuming the test was wrong.

The environment runs **SQLAlchemy 2.0.50**, whose ORM deduplicates single-entity
results by primary key. I proved the mechanism directly: running the exact join the
buggy code used against a 3-tag song returned **3 rows at the raw-SQL level** but only
**1 object** from `db.session.query(Song)...all()`:

```
Raw SQL join rows (DB level):     3
ORM query(Song).all() objects:    1
```

So the duplication is **latent** — the query really does fan out, but the installed
ORM version collapses the duplicate `Song` objects by identity before they reach the
caller. On an older SQLAlchemy, or if the query selected columns from the joined
table instead of whole entities, the same code would return the song three times,
exactly as the test comment predicts.

**How I found the root cause.**
Navigation path: `GET /songs/search` in `routes/songs.py` → `search_songs()` in
`services/search_service.py`. Reading the query, one clause stood out as the source
of the fan-out:

```python
db.session.query(Song)
  .outerjoin(song_tags, Song.id == song_tags.c.song_id)   # <-- fans out
  .filter(db.or_(Song.title.ilike(...), Song.artist.ilike(...)))
```

The confidence moment was realizing the join is not just the cause of the duplication
but is **entirely pointless**: `song_tags` appears nowhere in the `WHERE` clause (the
filter only touches `title`/`artist`) and nowhere in the `SELECT` (tags are loaded
separately via the `Song.tags` relationship inside `to_dict()`). It contributes
nothing except one output row per tag — a textbook unintended cartesian-style
fan-out.

**The root cause.**
`search_songs()` performed an `outerjoin` against the `song_tags` association table
that served no functional purpose. A song with N tags matches N rows of that join, so
the result set contained one copy of the song per tag (three copies for a 3-tag
song). The correctness of the visible output was accidentally being propped up by
SQLAlchemy 2.0's entity uniquing; the query itself was wrong.

**My fix and side-effect check.**
I removed the `.outerjoin(song_tags, ...)` line so the query filters `Song` directly
with no join, and dropped the now-unused `Tag, song_tags` import. I confirmed via the
compiled SQL that the statement is now a plain `SELECT ... FROM song WHERE ...` with
no join, so it can no longer fan out **regardless of ORM version** — the fix removes
the reliance on entity-uniquing rather than papering over it with `.distinct()`. Tags
still appear in each result because `to_dict()` reads them from the relationship,
which I verified is untouched. The full suite passes (**13 passed, 0 failed**),
including `test_search_returns_matching_songs` (tags still present) and all four
no-duplicate cases.

## Bug 4 — "Notified when a friend added my song to a playlist, but not when they rated it" (`notification_service.py`)

**How I reproduced it.**
There's no test for the notification side effect, so I wrote a script against an
in-memory database: a `sharer` shares a song, then a different user (`rater`) rates it
5/5 via `rate_song()`. Expected: the sharer gets one notification ("someone
interacted with your song"), the same way adding the song to a playlist notifies them.
Actual: zero notifications were created for the sharer:

```
Notifications for sharer after a friend rated their song: 0   (BUG)
```

**How I found the root cause.**
The report itself names the asymmetry — one interaction notifies, a sibling
interaction doesn't — so I opened `services/notification_service.py` and compared the
two functions side by side. `add_to_playlist()` ends with an explicit notify block:

```python
if song.shared_by != added_by_user_id:
    create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", body=...)
```

`rate_song()`, right below it, does all the same setup — loads the `song` and the
`rater`, saves the `Rating`, commits — and then simply `return rating`. The moment of
confidence was seeing that `rate_song()` has the exact ingredients the notify block
needs (`song.shared_by`, `rater.username`, `song.title`) already in local scope but
never calls `create_notification()` at all. It wasn't a wrong condition; the call was
absent.

**The root cause.**
`rate_song()` was missing the notification step entirely. It correctly created/updated
the `Rating` and returned it, but never told the song's original sharer that their
song had been rated. The module docstring ("Notifications are generated when friends
interact with a user's shared songs") describes the intended behaviour, and
`add_to_playlist()` implements it, but the rating path was never wired up to it.

**My fix and side-effect check.**
I added a notify block to `rate_song()` after the commit, mirroring
`add_to_playlist()` exactly: if `song.shared_by != user_id`, call
`create_notification(user_id=song.shared_by, notification_type="song_rated",
body=f"{rater.username} rated your song '{song.title}' {score}/5.")`. The
`shared_by != user_id` guard means users don't get notified for rating their own
songs — I verified both directions in the reproduction script: a friend's rating
produces exactly one `song_rated` notification, and the sharer rating their own song
produces none (count stays at 1). I placed the call after `db.session.commit()` so the
rating is persisted first, consistent with the playlist flow. Full suite: **13 passed,
0 failed** — no existing behaviour regressed.


=======
## Bugs fixed — summary

| # | Symptom | File | One-line cause | Fix |
|---|---------|------|----------------|-----|
| 1 | Listening streak keeps resetting | `streak_service.py` | Extra `and today.weekday() != 6` clause forced a reset every Sunday | Delete the clause |
| 2 | "Friends Listening Now" shows people from yesterday | `feed_service.py` | "Recent" window was 24h, so yesterday still counts as "now" | Shrink `RECENT_THRESHOLD` to 10 min |
| 3 | Same song shows up twice in search | `search_service.py` | Pointless `outerjoin(song_tags)` fans out one row per tag | Remove the join |
| 4 | Notified on playlist-add but not on rating | `notification_service.py` | `rate_song()` never called `create_notification()` | Add the notify block |
| 5 | Last song in a playlist never shows | `playlist_service.py` | Return statement sliced `songs[:-1]` | Return the full list |

---

# Root Cause Analysis

Each entry below records how I reproduced the bug, how I located the exact cause,
what the cause was, **why the fix actually resolves it**, and **what else I verified**
so the change is provably safe. All fixes were confirmed with `pytest` (13 tests) or a
targeted reproduction script whose output I read directly.

## Bug 1 — "My listening streak keeps resetting" (`streak_service.py`)

**How I reproduced it.** The symptom is date-dependent, so I reproduced it with fixed
dates instead of waiting for a real Sunday. Sequence: a user listens on Saturday
(streak → 1), then listens the very next day, Sunday. Expected streak = 2 (two
consecutive days); actual = 1. This is exactly what `test_streak_increments_on_sunday`
in `tests/test_streaks.py` encodes — it feeds `2024-06-15` (Sat) then `2024-06-16`
(Sun) into `update_listening_streak()` and asserts `2`. Against the original code that
assertion fails, which explains the "keeps resetting" report: the streak silently died
every Sunday.

**How I found the root cause.** I traced the path `POST /songs/<id>/listen`
(`routes/songs.py`) → `record_listening_event()` → `update_listening_streak()`. All the
arithmetic is in that one function, keyed off `days_since_last` (0 = same day, 1 =
consecutive, else = gap). Reading the consecutive-day branch, I saw a second, unrelated
condition welded onto it: `elif days_since_last == 1 and today.weekday() != 6:`.

**The root cause.** Python's `date.weekday()` returns `6` for Sunday. When the listen
fell on a Sunday, the branch evaluated `True and False → False`, control dropped to the
`else`, and the streak reset to 1 — even though the user *had* listened the day before.
The consecutiveness test itself was correct; the `weekday() != 6` term was spurious.

**The fix and why it works.** I removed the clause so the branch reads
`elif days_since_last == 1:`. This works because the three branches partition on
elapsed days: `== 0` holds, `== 1` increments, everything else resets. The bug was an
extra term that could turn a legitimate `== 1` case into the reset path one day a week;
removing it makes "consecutive day" the *only* thing that decides an increment, which
is the documented rule. Crucially, the term I deleted could only ever *force a reset*,
so removing it cannot introduce a *missed* reset — the genuine reset path (`else`, for
`days_since_last > 1`) is untouched.

**What else I verified.** I re-ran the full `tests/test_streaks.py` suite (5 tests) and
confirmed every neighbouring behaviour still holds: new user starts at 1, a normal
consecutive weekday still increments, two listens the same day don't double-count, and
a genuinely skipped day still resets to 1. That last one is the important safety check
— it proves I fixed the Sunday case without weakening real gap detection. Full project
suite stayed green (13/13).

## Bug 2 — "Friends Listening Now shows people from yesterday" (`feed_service.py`)

**How I reproduced it.** There's no feed test, so I wrote a script on an in-memory DB:
a user and a friend who are friends, and a single `ListeningEvent` for the friend
timestamped **20 hours ago** (yesterday, but within a day). Calling
`get_friends_listening_now(me.id)` returned that friend — a "listening now" feed
surfacing someone who last listened yesterday afternoon.

**How I found the root cause.** Path: `GET /feed/<id>/listening-now` (`routes/feed.py`)
→ `get_friends_listening_now()`. The query filters `ListeningEvent.listened_at >=
cutoff` where `cutoff = now - RECENT_THRESHOLD`, which correctly keeps events *newer*
than the cutoff — so the comparison direction was fine and the only remaining input was
the window size: `RECENT_THRESHOLD = timedelta(hours=24)`.

**The root cause.** The "recent" window was a full 24 hours, so anyone who listened any
time since this hour yesterday qualified as "listening now." The feed was never meant
to be a day-long history — that role belongs to `get_activity_feed()` further down the
file — it's meant to show who is *currently* playing something.

**The fix and why it works.** I set `RECENT_THRESHOLD = timedelta(minutes=10)`. This
works precisely *because* the query was already correct: `cutoff = now - 10min` means
the same `listened_at >= cutoff` filter now admits only events from the last ten
minutes, and a listen from 20 hours ago is far below that cutoff and is excluded. I
changed the constant that *defines* "recent," not the query — the minimal change that
fixes the reported behaviour.

**What else I verified.** I re-ran the script in both directions to make sure I hadn't
over-corrected: a 20-hour-old (yesterday) listen is now excluded (feed size 0), and a
genuinely recent 5-minute-old listen still appears (feed size 1). I confirmed
`get_activity_feed()` is untouched and deliberately does *not* use `RECENT_THRESHOLD`,
so the "full history" feed and the "now" feed stay semantically distinct — narrowing
one didn't silently narrow the other. The per-friend dedup logic still returns one
entry per friend. Full suite: no new failures.

## Bug 3 — "The same song keeps showing up twice in search" (`search_service.py`)

**How I reproduced it — and a caveat.** `tests/test_search.py` builds a song with three
tags and asserts it appears once. I ran the suite first to confirm the bug, but **all
five tests passed**. Rather than assume the test was wrong, I ran the query directly:
the raw SQL join returns **3 rows** for the 3-tag song, but `db.session.query(Song).all()`
returns **1 object**. The installed **SQLAlchemy 2.0.50** deduplicates single-entity
results by primary key, so the duplication never reaches the caller *in this
environment*. The bug is real but **latent** — on an older SQLAlchemy, or if the query
selected columns from the joined table, the song would come back three times.

**How I found the root cause.** Path: `GET /songs/search` (`routes/songs.py`) →
`search_songs()`. The query does `db.session.query(Song).outerjoin(song_tags, ...)`
then filters on title/artist. The join stood out as both the fan-out source and
entirely pointless: `song_tags` appears in neither the `WHERE` (which only touches
`title`/`artist`) nor the `SELECT` (tags are loaded separately via the `Song.tags`
relationship inside `to_dict()`).

**The root cause.** An `outerjoin` against the `song_tags` association table that served
no functional purpose. A song with N tags matches N join rows, producing one copy per
tag. The visible output was accidentally being propped up by SQLAlchemy's entity
uniquing; the query itself was wrong.

**The fix and why it works.** I removed the `.outerjoin(song_tags, ...)` line (and the
now-unused `Tag, song_tags` import). This works by construction: with no join the
statement is `SELECT ... FROM song WHERE title/artist LIKE ...`, which yields exactly
one row per matching song — so duplicates are impossible **regardless of ORM version**.
I deliberately chose to remove the join rather than paper over it with `.distinct()`,
because `.distinct()` would only mask the fan-out while still relying on the DB to
collapse rows; removing the cause is the robust fix.

**What else I verified.** I inspected the compiled SQL to confirm the `JOIN` is gone. I
re-ran `tests/test_search.py` (5 tests): the "matching songs" test confirms tags still
appear in each result (proving `to_dict()`'s relationship load is unaffected by
dropping the join), and all four no-duplicate cases pass. I kept the before/after
raw-SQL-vs-ORM measurement (3 rows → 1 row, now 1 row → 1 row) as evidence of the
mechanism. Full suite: 13/13.

## Bug 4 — "Notified when a friend added my song to a playlist, but not when they rated it" (`notification_service.py`)

**How I reproduced it.** No test covers the notification side effect, so I scripted it:
a `sharer` shares a song, a different `rater` rates it 5/5 via `rate_song()`. Expected:
the sharer gets one notification, the same way a playlist-add notifies them. Actual:
zero notifications were created for the sharer.

**How I found the root cause.** The report names the asymmetry directly, so I compared
the two sibling functions in `notification_service.py` side by side.
`add_to_playlist()` ends with an explicit notify block guarded by
`song.shared_by != added_by_user_id`. `rate_song()` right below it does all the same
setup — loads the `song` and `rater`, saves the `Rating`, commits — then simply
`return rating`. It had every ingredient the notify block needs (`song.shared_by`,
`rater.username`, `song.title`) already in local scope but never called
`create_notification()`.

**The root cause.** `rate_song()` was missing the notification step entirely. It
persisted the rating correctly but never told the song's sharer, even though the module
docstring promises "notifications are generated when friends interact with a user's
shared songs" and `add_to_playlist()` implements exactly that.

**The fix and why it works.** I added a notify block after the commit, mirroring
`add_to_playlist()`: if `song.shared_by != user_id`, call
`create_notification(user_id=song.shared_by, notification_type="song_rated",
body=f"{rater.username} rated your song '{song.title}' {score}/5.")`. This works
because it reuses the exact mechanism the working playlist path already uses —
`create_notification()` inserts a `Notification` row addressed to the sharer — so the
two interactions become symmetric. The `shared_by != user_id` guard reproduces the
playlist function's rule that you aren't notified about your own actions, and placing
the call *after* `db.session.commit()` means the `Rating` is durably saved before the
notification is created, matching the ordering in the playlist flow.

**What else I verified.** I ran the script in both directions: a friend's rating now
produces exactly one `song_rated` notification with the correct body, and the sharer
rating their *own* song produces none (the guard holds, count stays put). I also
reasoned about the re-rating path — `rate_song()` updates an existing `Rating` under the
`UniqueConstraint(user_id, song_id)`; the notify block sits after that branch, so both
first-time and repeat ratings notify. Full suite: 13/13, so the existing
`add_to_playlist()` notification and the rating-persistence behaviour both still work.

## Bug 5 — "The last song in a playlist never shows up" (`playlist_service.py`)

**How I reproduced it.** Two existing tests in `tests/test_playlists.py` pin this down,
so I ran them first to confirm red. `seed_playlist` builds a 5-song playlist
(Track 1–5) at positions 1–5. `test_playlist_returns_all_songs` asserts 5 songs come
back; `test_playlist_returns_songs_in_order` asserts the titles are Track 1–5. Both
failed — the call returned only 4 songs, dropping Track 5.

**How I found the root cause.** Path: `GET /playlists/<id>/songs`
(`routes/playlists.py`) → `get_playlist_songs()`. The query joins through
`playlist_entries` and orders by `position` ascending, so `songs` is the full,
correctly-ordered list — the bug had to be after the query. The return line was
`return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice was the moment of
confidence, and it also explains why the *third* test
(`test_empty_playlist_returns_empty_list`) still passed: `[][:-1]` is `[]`, so the
truncation is invisible for empty playlists and only bites once there's at least one
song.

**The root cause.** The comprehension iterated over `songs[:-1]` instead of `songs`.
`[:-1]` returns everything except the last element, so the highest-`position` song in
every non-empty playlist was discarded before serialization. The database query was
correct; the truncation happened purely in the return statement, contradicting the
function's own docstring ("returns all songs in the playlist").

**The fix and why it works.** I changed `songs[:-1]` to `songs`. This works because the
slice was the *only* thing limiting the result — there is no other `LIMIT`, filter, or
truncation in the function — so iterating the complete list serializes exactly the rows
the (already-correct) ordered query returned. The `ORDER BY position` is preserved, so
songs still come back in playlist order.

**What else I verified.** Both previously-failing tests flip green: 5 songs returned,
and the exact Track 1–5 order asserted (confirming I preserved ordering, not just
count). The empty-playlist test still passes, so the change is safe for the zero-song
case. I ran the **full** suite afterwards — 13 passed, 0 failed — confirming no
regression anywhere else.
>>>>>>> Stashed changes
