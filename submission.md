# Mixtape — Bug Hunt Submission

## AI usage

I used Claude throughout this project, mostly for codebase orientation and
for narrowing down root causes I'd already partially identified — not for
diagnosing bugs cold.

**Orientation (before touching any issue):** I had Claude read the main
files and explain them individually (`app.py`, `models.py`, each route
module) and then trace one full data flow end-to-end (adding a song to a
playlist → notification). I also asked targeted questions about SQLAlchemy
mechanics I wasn't confident about — what `db.relationship()` actually does
versus a plain `db.Column`, why the `friends` relationship needs explicit
`primaryjoin`/`secondaryjoin` (self-referential many-to-many), and what
`db.session.commit()` does versus `db.session.add()`. This built the mental
model in the codebase map before I looked at any of the 5 issues, per the
brief's instruction.

**During investigation:** for each bug, I'd point Claude at a specific
function and ask it to compare two similar code paths line by line (e.g.
`add_to_playlist()` vs. `rate_song()` in `notification_service.py` for issue
#4) or explain a specific Python behavior I'd already narrowed the bug down
to (e.g. what `date.weekday()` returns for each day, once I'd already spotted
`today.weekday() != 6` as the suspicious line for issue #1). I did not ask it
to find bugs by reading the codebase cold — every diagnosis started from a
specific line or function I'd already flagged as suspicious.

**Where I had to verify (or where the AI's first guess was wrong):** while
investigating issue #3 (duplicate search results), Claude's initial
hypothesis — that the `outerjoin` against `song_tags` with no `.distinct()`
would fan out into duplicate rows for any song with 2+ tags — turned out to
be **wrong**. Actually running `search_songs()` against a 3-tag song returned
only 1 result, not 3. Rather than accept the plausible-sounding theory, we
ran it and found it didn't hold, and left issue #3 unsolved rather than
"fixing" a cause that wasn't real. Similarly for issue #2, an initial theory
that `listened_at` losing timezone info on its SQLite round-trip (confirmed
true) would cause stale entries to leak past the 24-hour cutoff didn't
reproduce in a direct boundary test — the filter behaved correctly in that
test despite the tz inconsistency being real. Both of these are documented
below as genuine, unresolved attempts rather than being papered over.

**Where a hypothesis was confirmed by running it:** for issue #5, Claude
predicted from reading `add_to_playlist()`'s use of `playlist.songs.append(song)`
that it would fail to populate the `position`/`added_by` columns on
`playlist_entries` (both `NOT NULL`, no default) and crash. This was verified
live — it does crash, with exactly the predicted `IntegrityError`. That
finding isn't one of the 5 listed issues, so I didn't fix it, but it's noted
in the issue #5 write-up since it affects how that fix can be tested.

Every fix in this document followed the same loop: read the suspicious code
→ form a hypothesis (sometimes with Claude's help articulating it) → verify
by actually running the code with controlled inputs → only then change
anything.

## Main files and what they do

- **`app.py`** — Flask application factory (`create_app`). Configures the SQLAlchemy
  database URI (defaults to local `sqlite:///mixtape.db`), initializes the `db`
  extension, registers the four blueprints (`songs`, `playlists`, `users`, `feed`)
  under their URL prefixes, and calls `db.create_all()` on startup. This is the
  only place blueprints get wired up — nothing else touches `app.config`.

- **`models.py`** — All SQLAlchemy models and the raw association tables:
  - `User` — has `listening_streak` / `last_listened_at` (streak state lives directly
    on the user row, not in a separate table), and a self-referential many-to-many
    `friends` relationship via the `friendships` table. Friendship is modeled as two
    directed rows (no evidence the app enforces symmetry when creating a friendship —
    worth checking if that's ever a problem).
  - `Song` — shared by exactly one user (`shared_by`), with `ratings`,
    `listening_events`, and a `tags` many-to-many via `song_tags`.
  - `ListeningEvent` — one row per (user, song, timestamp) listen. This is the
    source of truth for both the streak logic and the friends feed.
  - `Rating` — one row per (user, song), enforced by a unique constraint
    (`unique_user_song_rating`), score 1–5.
  - `Playlist` — songs live in `playlist_entries`, a many-to-many table that also
    carries `position` (explicit ordering, not insertion order), `added_by`, and
    `added_at`. This is the same "richer join table" pattern as `song_tags`.
  - `Notification` — a generic typed message (`notification_type` + `body`) per
    user, with a `read` flag. There's no polymorphic link back to the thing that
    caused it — the triggering context is only preserved in the free-text `body`.

- **`routes/`** — Four blueprints, each a thin HTTP layer:
  - `feed.py` — friends-listening-now, activity feed.
  - `playlists.py` — create, get-by-id, list songs, add song.
  - `songs.py` — search, get-by-id, rate, listen.
  - `users.py` — get user, get streak, get/mark notifications.

  Every route follows the same shape: pull params out of `request`, do minimal
  presence validation (400 if missing), call exactly one service function, and
  translate a `ValueError` raised by the service into a 404 or 400 JSON response.

- **`services/`** — All business logic, one module per feature area:
  - `feed_service.py` — `get_friends_listening_now()` (last-24h events per friend, deduped to most recent) and `get_activity_feed()` (most recent N events,
    no recency filter).
  - `notification_service.py` — the one place `Notification` rows get created
    (`create_notification()`). Of the two user-facing actions this module
    exposes, only `add_to_playlist()` actually calls it; `rate_song()` upserts
    a `Rating` row but never notifies the song's sharer. Also owns
    notification retrieval (`get_notifications()`) and `mark_as_read()`.
  - `playlist_service.py` — `create_playlist()`, `get_playlist_songs()` (orders
    by `playlist_entries.position`), `get_playlist()`, `get_user_playlists()`.
  - `search_service.py` — `search_songs()` (title/artist `ILIKE` match, joined
    with tags) and `get_song()`.
  - `streak_service.py` — `record_listening_event()` creates a `ListeningEvent`
    and calls `update_listening_streak()`, which compares calendar dates
    (`today` vs `last_listened_at.date()`) to decide whether to no-op, increment,
    or reset the streak to 1.

- **`seed_data.py`** — Populates the DB with sample users/songs/playlists for
  manual testing and running the app locally.

- **`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py` —
  one test file per service area with known open issues. No test file yet for
  `feed_service.py` or `notification_service.py`.

## Data flow — user adds a shared song to a playlist (triggers a notification)

1. Client sends `POST /playlists/<playlist_id>/songs` with `{song_id, added_by}`,
   handled by `routes/playlists.py`.
2. `routes/playlists.py:add_song()` does presence validation on `song_id` and
   `added_by`, then calls `services/notification_service.py:add_to_playlist(
   playlist_id, song_id, added_by)`. The route itself never touches `Playlist`
   or `Song` directly — even though this is nominally a "playlist" action, the route delegates to `notification_service.py`, not `playlist_service.py`, because the notification module owns the *combined* "add + notify" operation.
3. Inside `add_to_playlist()` in `services/notification_service.py`:
   - Looks up the `Song`, the adding `User`, and the `Playlist` (all defined
     in `models.py`) — raises `ValueError` (→ 400 at the route) if any is
     missing.
   - If the song isn't already in `playlist.songs`, appends it and commits.
     This mutates the `playlist_entries` association table (`models.py`)
     directly through the ORM relationship.
   - **Only if** `song.shared_by != added_by_user_id` (i.e., you didn't add your
     own song), it calls `create_notification()`, also in
     `services/notification_service.py`, with
     `notification_type="song_added_to_playlist"` and a body string built from
     the adder's username, the song title, and the playlist name.
4. `create_notification()` in `services/notification_service.py` just
   constructs a `Notification` row (`models.py`) and commits — it has no
   awareness of *why* it's being called; the calling function is responsible
   for deciding when a notification is warranted and for wording the message.
5. The recipient later fetches notifications via `GET
   /users/<user_id>/notifications`, handled by `routes/users.py:notifications()`,
   which calls `services/notification_service.py:get_notifications()` — that
   just queries `Notification` by `user_id` (optionally filtered to unread)
   ordered newest first.

Contrast with rating a song (`POST /songs/<id>/rate`, handled by `routes/songs.py:rate()`, which calls `services/notification_service.py: rate_song()`): it upserts a `Rating` row (`models.py`) but **never calls `create_notification()`** — the sharer is not notified when their song is rated, even though the module clearly has the machinery to do so.

## Patterns noticed

- **Strict routes → services layering.** Every blueprint function is I/O
  glue only (parse request, call one service function, shape the response).
  All conditionals and DB queries live in `services/`. This means every bug report that describes wrong *behavior* (vs. a wrong HTTP status/shape) should be chased into the matching service file, not the route.
- **Errors as `ValueError`, not custom exception types.** Every service raises a plain `ValueError` with a human-readable message for "not found" or invalid-input cases; every route catches `ValueError` and maps it to 400 or 404. Consistent, but means the route can't distinguish "not found" from "invalid input" by exception type — only by call site.
- **Association tables double as lightweight metadata stores.** `song_tags`  is a plain M2M table, but `playlist_entries` and `friendships` extend the
  pattern to carry ordering (`position`) or directional identity — a hint
  that "just a join table" assumptions (e.g. in `search_service`'s
  `song_tags` join) may not hold once a table like this grows a real
  ordering/metadata column, and can also cause row fan-out during joins that
  aren't `.distinct()`'d.
- **Cross-service imports happen locally, not at module load.** In
  `services/notification_service.py`, `add_to_playlist()` imports `Playlist`
  (from `models`) and `get_playlist_songs` (from `playlist_service`) *inside*
  the function body, rather than at the top of the file like every other
  import in the project.
- **Docstrings describe intended behavior, which is a good bug-hunting
  signal.** Several service functions have docstrings that spell out the
  exact contract (e.g. "returns all songs in the playlist", "streak resets
  to 1 if a day is skipped") — comparing the docstring's claim against the
  actual code below it is often enough to spot a bug without needing to run
  anything.

## Chosen bugs and reproduction

I'm fixing **Issues #1, #4, and #5** — all three reproduced as following:

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** The real HTTP endpoint (`POST /songs/<id>/listen`)
can't be used to reproduce this on demand, because `update_listening_streak()`
derives `now` from the real system clock (`datetime.now()`), and the bug only
manifests when "today" is a Sunday — the actual day this was tested was not a
Sunday. So I called the service function directly with a controlled `now`,
via `flask shell`:

```python
from datetime import datetime, timezone
from app import db
from models import User
from services.streak_service import update_listening_streak

user = User.query.filter_by(username="nova").first()

# Precondition: user listened yesterday (Saturday)
user.listening_streak = 5
user.last_listened_at = datetime(2026, 7, 4, 12, 0, tzinfo=timezone.utc)  # Saturday
db.session.commit()

# Trigger: simulate listening again the next day, which is a Sunday
update_listening_streak(user, datetime(2026, 7, 5, 12, 0, tzinfo=timezone.utc))
print(user.listening_streak)  # -> 1 (bug: should be 6, a normal +1 increment)
```
As a control, I repeated the identical setup with a Monday→Tuesday pair
instead of Saturday→Sunday, and the streak incremented correctly to `6`. This
isolates the failure to the Sunday transition specifically, not a general
off-by-one in the day-gap calculation.

**How I found the root cause:** Started at `routes/songs.py:listen()`, which
calls `streak_service.record_listening_event(user_id, song_id)`. That
function creates the `ListeningEvent` and then delegates all the actual
streak math to `update_listening_streak(user, now)`. Reading that function's
three branches (no-op / increment / reset), the increment branch stood out:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
```
Every other branch depends only on `days_since_last`; this is the one branch
with an extra, unexplained condition on the day of the week. That asymmetry —
and the fact that `weekday() == 6` is Sunday in Python — was the moment I was
confident this was the actual cause, not just a suspicious area. I confirmed
it by running the reproduction above before making any change.

**The root cause:** In `update_listening_streak()`
(`services/streak_service.py`), the branch that increments the streak for a
consecutive-day listen required both `days_since_last == 1` **and**
`today.weekday() != 6`. Python's `date.weekday()` returns `6` for Sunday, so
`!= 6` evaluates to `False` on Sundays. When a user listened yesterday and
listens again today, and today happens to be a Sunday, the `elif` condition
as a whole is `False` even though the "listened yesterday" precondition is
satisfied — so execution falls through to the `else` branch and the streak is
reset to `1` instead of incremented. Every other day of the week correctly
takes the increment branch; Sunday is the only day where a legitimate
consecutive-day listen gets treated as if a day had been skipped.

**Fix and side-effect check:** Removed the `and today.weekday() != 6` clause,
so the increment branch is now just `elif days_since_last == 1:` — a daily
streak has no legitimate reason to treat any particular day of the week
differently. After the fix, I re-ran four cases directly against
`update_listening_streak()`:
- Saturday → Sunday (the bug case): streak now goes `5 → 6` (was `5 → 1`)
- Monday → Tuesday (control): still correctly `5 → 6`
- Same-day listen (`days_since_last == 0`): still correctly a no-op (`5 → 5`)
- 3-day gap (`days_since_last > 1`): still correctly resets to `1`

All four branches behave correctly, confirming the fix addresses the Sunday
case without changing the no-op or reset behavior. I also checked
`get_streak()` (the only other function touching `listening_streak`) — it's a
pure read with no date logic, so it's unaffected by this change.

### Issue #4 — Rating a song doesn't notify the sharer

**How I reproduced it:** Reproducible directly through the real HTTP API, no
date manipulation needed. Using the seeded DB:

```bash
# darius shares "Golden Hour"; sharer_id = darius's user id
curl http://127.0.0.1:5000/users/<darius_id>/notifications
# -> {"notifications": [], "count": 0}

# nova (a different user) rates darius's song
curl -X POST http://127.0.0.1:5000/songs/<golden_hour_song_id>/rate \
  -H "Content-Type: application/json" \
  -d '{"user_id": "<nova_id>", "score": 5}'
# -> 201, rating created

curl http://127.0.0.1:5000/users/<darius_id>/notifications
# -> {"notifications": [], "count": 0}   <-- unchanged, no notification created
```
As a control, I repeated the same before/after check around
`POST /playlists/<id>/songs` (adding the same song to a playlist as a
different user) and confirmed *that* action does create a notification for
the sharer — proving the notification machinery itself works, and the gap is
specific to the rating action.

**How I found the root cause:** Started at `routes/songs.py:rate()`, which
calls `notification_service.rate_song()`. Since `notification_service.py`
already has a *working* notification flow in the same file —
`add_to_playlist()` — I compared the two functions line by line rather than
reading `rate_song()` in isolation. `add_to_playlist()` ends with:
```python
if song.shared_by != added_by_user_id:
    create_notification(user_id=song.shared_by, notification_type="song_added_to_playlist", ...)
```
`rate_song()` has every piece this needs already in scope — `song` (fetched
at the top), `rater` (the `User` object, fetched right after), and `user_id`
(the parameter) — but its final lines are just:
```python
db.session.commit()
return rating
```
No equivalent `create_notification()` call, and no equivalent guard. That was
the confirming moment: this isn't a subtle logic error, it's a step that was
never written, even though every value the step would need is already
sitting in local variables.

**The root cause:** `rate_song()` in `services/notification_service.py`
persists the `Rating` row and returns — it never calls `create_notification()`.
This is architectural, not a typo: the function was written to only handle
the rating's own persistence, and whoever wrote `add_to_playlist()`'s
"notify the sharer" step never added the equivalent call to `rate_song()`.
The notification system itself (`create_notification()`, `get_notifications()`)
works correctly; it's just never invoked from this one code path.

**Fix and side-effect check:** Added the same notify-the-sharer step used in
`add_to_playlist()`, placed after the existing `db.session.commit()`:
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```
I used a distinct `notification_type` (`"song_rated"` vs.
`"song_added_to_playlist"`) so the two notification kinds remain
distinguishable to any future code that filters by type. After the fix, I
re-verified three cases directly against `rate_song()`:
- A different user rates the song → notification count `+1`, with the
  expected body text.
- The sharer rates their **own** song → notification count unchanged (mirrors
  `add_to_playlist()`'s existing self-action guard).
- The same user re-rates the song (updating their existing `Rating` row
  instead of creating a new one) → still notifies again, consistent with
  `add_to_playlist()` notifying on every call regardless of whether the
  underlying row was inserted or already existed.

I also re-ran the existing rating flow to confirm the `Rating` upsert
behavior (new vs. updated score) is completely unchanged — the fix only adds
a call after the existing `commit()`, it doesn't touch the rating logic
itself.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it:** Used one of the playlists already populated by
`seed_data.py` ("Late Night Vibes", 7 songs inserted directly into
`playlist_entries` with explicit `position` values), rather than adding songs
through the API — see the note below on why.

```bash
sqlite3 instance/mixtape.db \
  "SELECT COUNT(*) FROM playlist_entries WHERE playlist_id = '<late_night_vibes_id>';"
# -> 7

curl http://127.0.0.1:5000/playlists/<late_night_vibes_id>/songs
# -> {"songs": [...], "count": 6}   <-- one song short of the 7 actually stored
```
6 returned vs. 7 stored confirms the last song (by `position`) is being
dropped somewhere between the DB and the response.

**How I found the root cause:** Started at `routes/playlists.py:get_songs()`,
which calls `playlist_service.get_playlist_songs()`. That function's
docstring says, in its `Note:` section, *"This function returns all songs in
the playlist"* — a direct, explicit claim about behavior. Reading the
function body, the query itself (join on `playlist_entries`, filter by
`playlist_id`, order by `position` ascending) looks correct and matches the
docstring's description of ordering. The very last line is where the
docstring's claim breaks:
```python
return [song.to_dict() for song in songs[:-1]]
```
`songs[:-1]` is a slice that drops the last element of whatever list precedes
it. Since `songs` was already correctly ordered by position, this always
discards the highest-position (i.e. most recently added) song, regardless of
how many songs are in the playlist. That contradiction — a function whose own
docstring says "all songs" immediately followed by code that provably returns
one fewer — was the confirming moment.

**The root cause:** `get_playlist_songs()` in `services/playlist_service.py`
correctly queries and orders every song in the playlist, but its return
statement slices the result with `songs[:-1]` before converting to dicts.
This unconditionally drops the last item of the ordered list — i.e. the song
with the highest `position` value — from every playlist, no matter how many
songs it contains.

**Fix and side-effect check:** Changed the return statement from
`songs[:-1]` to `songs`, so all queried songs are returned. I checked three
cases directly against `get_playlist_songs()` after the fix:
- A 7-song seeded playlist: now returns all 7 (previously 6).
- An empty (0-song) playlist: still correctly returns `0` — this boundary was
  never actually broken, since slicing an empty list with `[:-1]` also
  produces an empty list, so this case looked fine both before and after.
- A 1-song playlist: now correctly returns `1`. This is actually the most
  severe instance of the original bug — `[:-1]` applied to a single-item list
  produces an empty list, so a playlist with exactly one song would have
  appeared **completely empty** via the API, not just "missing its last
  song." This wasn't mentioned in the original bug report but follows
  directly from the same root cause.

I also re-checked `get_playlist()` and `get_user_playlists()` (the other two
read functions in this file) — neither touches `playlist_entries` or does any
slicing, so they're unaffected by this change.
