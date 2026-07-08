# Mixtape — Bug Hunt Submission


**Debugging Issue #1 (streak):** I ran the existing test suite myself to find the failing test. Once I had `test_streak_increments_on_sunday` failing, I pasted the `update_listening_streak` function and asked Claude to walk through what `.weekday()` returns for Sunday and how that interacted with the `elif` condition. This confirmed my own reading of the code — the day-of-week check was the problem — rather than Claude finding it for me from scratch.

**Debugging Issue #4 (notifications):** Claude suggested comparing `rate_song()` against the working `add_to_playlist()` function line by line, per the hint in the brief that this bug was architectural. That comparison made it obvious `rate_song()` was missing the entire notification step. I applied the fix, but hit a snag afterward — my server didn't pick up the change because Flask's dev server wasn't running with auto-reload (debug mode was off). Claude helped me figure that out from the server logs; restarting the server was the actual fix, not another code change.

**Debugging Issue #5 (playlist):** I read `get_playlist_songs()` myself and noticed the `songs[:-1]` slice looked suspicious against the docstring claiming "returns all songs." I confirmed with Claude that a `[:-1]` slice drops the last list element, which matched my test failure exactly.

**Where AI got it wrong:** Early on, Claude assumed Issue #3 (search duplicates) was already reproduced based on an earlier successful search for a single-tag song — which wasn't actually a valid test of the bug condition (that song only had 1 tag, not multiple). When I later ran the actual test suite, all search tests passed, including the one specifically designed to catch Issue #3. Claude then went back and actually ran the exact `search_service.py` code against a 3-tag song directly to check, and confirmed SQLAlchemy's ORM automatically de-duplicates joined query results by primary key — so the "obvious" duplicate-via-join theory didn't hold up under real testing. This was a useful reminder that AI's first guess at a root cause isn't reliable until it's actually verified against the code and real data, which is why I ended up swapping Issue #3 out for Issue #4 as my third bug.

---

## Codebase Map

### Main files and their roles

| File | Responsibility |
|---|---|
| `app.py` | Flask app factory. Registers 4 blueprints (`songs`, `playlists`, `users`, `feed`) and sets up the SQLAlchemy `db` instance. |
| `models.py` | All SQLAlchemy models: `User`, `Song`, `Tag`, `Playlist`, `Rating`, `ListeningEvent`, `Notification`, plus association tables for friendships, song tags, and playlist entries (with `position`, `added_by`, `added_at`). |
| `routes/songs.py` | Search, song detail, rating, and listening-event endpoints. Thin — parses request data and delegates to services. |
| `routes/playlists.py` | Playlist creation, detail, listing songs, adding songs. |
| `routes/users.py` | User profile, streak, and notification endpoints. |
| `routes/feed.py` | "Listening now" and "activity" feed endpoints. |
| `services/streak_service.py` | Owns streak increment/reset logic and recording listening events. |
| `services/feed_service.py` | Builds the friends-listening-now and activity feeds. |
| `services/search_service.py` | Song search by title/artist. |
| `services/notification_service.py` | Creates and retrieves notifications; also owns `add_to_playlist` and `rate_song` (business logic lives here, not in routes). |
| `services/playlist_service.py` | Playlist creation and retrieval, including ordered song lists. |

### Pattern noticed

Every route is a thin wrapper: parse the request → call a service function → catch `ValueError` → return JSON with an appropriate status code. All real logic (queries, business rules) lives in `services/`, matching what the README describes. Practically, this means the route file is almost never where a bug actually lives — you have to trace into the service function it calls.

Another pattern: side effects are sometimes bundled into a single service function rather than split out. For example, `notification_service.add_to_playlist()` both adds the song to the playlist *and* creates the notification in the same function call. This matters when tracing "why didn't X happen" — the answer isn't always in a dedicated function for X, which turned out to be directly relevant to Issue #4.

### Data flow — adding a song to a playlist (triggers a notification)

1. Client sends `POST /playlists/<playlist_id>/songs` with `song_id` and `added_by` in the body.
2. `routes/playlists.py::add_song()` validates both fields are present, then calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. Inside `add_to_playlist`:
   - Looks up the `Song`, `User` (adder), and `Playlist` — raises `ValueError` (→ 400 response) if any is missing.
   - If the song isn't already in `playlist.songs`, appends it and commits.
   - If the adder isn't the song's original sharer, calls `create_notification(...)` to notify `song.shared_by`.
4. `create_notification` builds a `Notification` row and commits it.
5. Route returns `{"message": "Song added to playlist"}, 201`.

---

## Bug Fixes

### Issue #1: Listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/test_streaks.py -v`. `test_streak_increments_on_sunday` failed — it simulates a user listening on Saturday (streak becomes 1), then on Sunday (expected streak of 2, since Sunday follows Saturday). Actual result: streak stayed at 1.

**How I found the root cause:** Traced from `record_listening_event()` in `streak_service.py`, which calls `update_listening_streak()`. That function branches on `days_since_last`. The `elif` branch that increments the streak for a consecutive day had an extra condition: `and today.weekday() != 6`. I confirmed `datetime.weekday()` returns `6` for Sunday, meaning that condition is `False` specifically on Sundays — which excludes an otherwise-valid consecutive day from incrementing and sends it to the `else` branch instead.

**The root cause:** The `elif` condition required both `days_since_last == 1` *and* `today.weekday() != 6`. On a Sunday, the second condition is always false regardless of whether the listen was actually consecutive, so the streak incorrectly resets to 1 instead of incrementing — even though the docstring's own stated rule ("listened yesterday → increment by 1") makes no exception for Sundays.

**My fix and side-effect check:** Removed `and today.weekday() != 6` from the `elif` condition, leaving only the `days_since_last == 1` check. Ran the full streak test suite afterward — all 5 tests pass, including the non-Sunday consecutive-day and same-day tests, confirming the fix didn't affect those paths.

---

### Issue #4: No notification when a friend rates my song

**How I reproduced it:** Had one user (nova) rate a song shared by a different user (darius) via `POST /songs/<song_id>/rate`. The rating saved successfully (confirmed by the response with the correct score), but checking the sharer's notifications via `GET /users/<sharer_id>/notifications` returned `count: 0` — no notification was created.

**How I found the root cause:** Compared `rate_song()` to `add_to_playlist()` in `notification_service.py`, since both functions represent a friend interacting with a song I shared, and `add_to_playlist()` is known to work correctly. `add_to_playlist()` ends with a check (`if song.shared_by != added_by_user_id`) that calls `create_notification(...)`. `rate_song()` performs its core action — saving or updating the `Rating` — and returns immediately after `db.session.commit()`, with no equivalent notification step at all.

**The root cause:** This isn't a broken condition or typo — the notification-creation step is simply absent from `rate_song()`. The function was written to persist the rating but never had the corresponding "notify the sharer" logic added, even though that pattern already existed and worked in the sibling function `add_to_playlist()`.

**My fix and side-effect check:** Added a notification block to `rate_song()`, mirroring the existing pattern from `add_to_playlist()` — checking that the rater isn't the song's own sharer before calling `create_notification()` with a `song_rated` type. Verified by re-running the reproduction: rating a song as a different user now produces a `song_rated` notification for the sharer with the correct body text. Also confirmed the rating itself still saves/updates correctly, so the fix didn't disturb the rating logic itself — only added the missing step after it. (Note: had to restart the local Flask server for the fix to take effect, since debug/auto-reload was off.)

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Ran `pytest tests/test_playlists.py -v`. `test_playlist_returns_all_songs` failed — a playlist seeded with 5 songs returned only 4 from `get_playlist_songs()`. `test_playlist_returns_songs_in_order` failed the same way, missing "Track 5" specifically, the last song by position.

**How I found the root cause:** Read `get_playlist_songs()` in `playlist_service.py` top to bottom. The query itself (join, filter by playlist ID, order by position ascending) looked correct. The issue was in the return statement: `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element of the `songs` list before converting to dicts. This also directly contradicted the function's own docstring, which states "This function returns all songs in the playlist" — a clear sign the slice wasn't intentional.

**The root cause:** The query correctly fetches and orders every song in the playlist, but the final list comprehension iterates over `songs[:-1]` instead of `songs`, silently excluding whichever song is last in position order — regardless of which song that happens to be.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs` so all queried songs are returned. Verified with the full playlist test suite: all 3 tests pass, including the empty-playlist case (`test_empty_playlist_returns_empty_list`), confirming the fix doesn't break behavior when there are zero songs to return.

## AI Usage

I used Claude throughout this project for codebase navigation and debugging support, not for writing fixes blind.

**Orientation:** I gave Claude the contents of `app.py`, `models.py`, and all the `routes/`/`services/` files and asked it to summarize what each file was responsible for, and to trace the data flow for adding a song to a playlist (route → service → notification side effect). This helped me build the codebase map faster than reading every file cold, but I read the actual files myself before trusting the summary.