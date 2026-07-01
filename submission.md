# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project for codebase navigation and debugging support, not for generating fixes from scratch:

- Asked it to explain what each service file was responsible for before opening any issue, to build a mental model of the app (models, routes, services layers).
- Used it to trace the two example call chains from the README (rating a song, viewing a playlist) across routes and services before touching any bug.
- For each bug, I reproduced the behavior myself first using a Python shell with real seeded data, comparing expected vs. actual output, before looking for a fix.
- Once I confirmed a bug was real, I asked Claude to help me read the specific suspicious line or condition I'd already found (e.g. the `weekday() != 6` check, the `[:-1]` slice, the missing `create_notification()` call) and explain why it produced the observed behavior.
- I verified every fix myself by rerunning the reproduction steps and the relevant pytest file before committing, rather than trusting the explanation alone.

## Codebase Map

**Main files:**
- `models.py` — 7 models: User, Tag, Song, ListeningEvent, Rating, Playlist, Notification, plus 3 association tables. `playlist_entries` carries `position`, `added_by`, `added_at` — playlist order is explicit, not insertion order.
- `app.py` — Flask app factory; registers 4 blueprints (songs, playlists, users, feed) under url prefixes matching their names.
- `routes/` — thin layer: parses request data, calls a service function, formats the JSON response.
- `services/` — all business logic; each function does an existence check via `db.session.get()` and raises `ValueError` before doing anything else, which routes catch and return as 4xx errors.

**Data flow — rating a song:**
`POST /songs/<song_id>/rate` → `routes/songs.py::rate()` reads `user_id`/`score` from the JSON body → `notification_service.rate_song()` validates the score range, looks up the song and user, upserts a `Rating` row (one per user/song pair via a unique constraint), commits, and returns the rating. No notification was originally created in this path (see Issue #4).

**Data flow — adding a song to a playlist (contrast):**
`POST /playlists/<id>/songs` → `routes/playlists.py::add_song()` → `notification_service.add_to_playlist()` — appends the song to the playlist, commits, then explicitly calls `create_notification()` for the song's original sharer.

**Pattern noticed:** the two flows above are structurally similar (both live in `notification_service.py`, both check "was it me who did this?") but only one of them notified. That was the crux of Issue #4.

## Issues Read

All 5 read from the course brief table:
1. Listening streak keeps resetting — `streak_service.py`
2. Friends Listening Now shows people from yesterday — `feed_service.py`
3. Same song shows up twice in search — `search_service.py`
4. Missing notification on rating (vs. working notification on playlist add) — `notification_service.py`
5. Last song in a playlist never shows up — `playlist_service.py`

---

## Root Cause Analysis Entries

### Issue #1 — My listening streak keeps resetting

**How I reproduced it:** Using a Python shell with Flask app context, I called `update_listening_streak(user, saturday)` then `update_listening_streak(user, sunday)` on a real seeded user (darius), simulating two consecutive days landing on a Saturday→Sunday boundary. Expected the streak to go 1→2 since these are consecutive days. Actual result: it stayed at 1 both times — the Sunday call should have incremented but didn't.

**How I found the root cause:** Traced from `record_listening_event()` into `update_listening_streak()` in `streak_service.py`, which handles the day comparison logic. Read the increment branch and found a second condition ANDed onto the day-gap check.

**The root cause:** The streak only incremented when `days_since_last == 1 AND today.weekday() != 6`. Python's `weekday()` returns 6 for Sunday, so this extra condition excluded Sundays from ever counting as a valid consecutive-day increment. Any streak update landing on a Sunday fell through to the `else` branch and incorrectly reset to 1 instead of incrementing.

**My fix and side-effect check:** Removed `and today.weekday() != 6`, leaving `days_since_last == 1` as the sole check. Verified with `pytest tests/test_streaks.py -v` — all 5 tests pass, including same-day (no double-count), skipped-day (resets to 1), and Sunday (now correctly increments).

---

### Issue #4 — Missing notification on rating

**How I reproduced it:** Using a Python shell with app context, checked `get_notifications(song.shared_by)` count for the sharer of "Midnight Drive" before and after calling `rate_song()` as a different user (darius). The rating itself succeeded (a real Rating row was created), but the notification count stayed at 1 both before and after — no new notification was created for the song's sharer, even though rating a song should notify them the same way adding it to a playlist does.

**How I found the root cause:** Compared `add_to_playlist()` and `rate_song()` side by side in `notification_service.py`, since both represent a friend interacting with a song someone else shared. `add_to_playlist()` calls `create_notification()` after committing its change; `rate_song()` had no equivalent call anywhere in its body.

**The root cause:** `rate_song()` saves the `Rating` to the database but never calls `create_notification()`. The notification-creation step simply doesn't exist in this function's code path, unlike the structurally similar `add_to_playlist()`, which does call it. This isn't a typo — it's a missing step in the function, likely omitted when the feature was originally built.

**My fix and side-effect check:** Added a `create_notification()` call after the rating commits, following the same pattern as `add_to_playlist()` (only notify if the rater isn't the song's own sharer). Verified via Python shell: notification count went from 1 to 2 after a different user rated the song. Also re-ran the rating flow with the sharer rating their own song to confirm no notification was created in that case, matching the "don't notify yourself" pattern.

---

### Issue #5 — Last song in playlist never shows up

**How I reproduced it:** Using a Python shell with app context, compared the real song count on a seeded playlist ("Late Night Vibes") via the `playlist.songs` relationship (7 songs) against the count returned by `get_playlist_songs()` (6 songs). Comparing the actual vs. returned titles showed "Free Throws" — the song with the highest `position` value — was missing entirely from the returned results.

**How I found the root cause:** Traced `get_playlist_songs()` in `playlist_service.py`. It queries songs joined to `playlist_entries`, ordered ascending by `position`, then returns a list comprehension over `songs[:-1]` instead of `songs`. Confirmed this by comparing the actual song count on a seeded playlist against the count returned by the function — the missing song was always the one with the highest `position` value.

**The root cause:** After correctly querying and ordering all songs in the playlist by position, the function sliced the final list with `[:-1]`, which drops the last element of any list. This silently removed the last-added song from every playlist's results, regardless of playlist size.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs`, returning the full ordered list. Verified with `pytest tests/test_playlists.py -v` — all 3 tests pass, including `test_playlist_returns_all_songs` (previously failing at 4 instead of 5) and `test_playlist_returns_songs_in_order`, confirming ordering wasn't affected by the fix. Also confirmed empty playlists still return an empty list without error.