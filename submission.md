# Mixtape — Bug Hunt Submission

## AI Usage

I used Claude as a second set of eyes while I read through the codebase and debugged — mainly to explain things I'd already found, not to find bugs for me.

**Orientation:** I fed it the route and service files and asked it to explain what each one did and to trace how adding a song to a playlist ends up creating a notification. That saved me some time piecing the call chain together, but I still went through the files myself before writing the map in my own words.

**Issue #1 (streak):** I found the failing test myself (`test_streak_increments_on_sunday`) and read through `update_listening_streak`. I asked Claude to confirm what `.weekday()` returns for Sunday since I wasn't 100% sure, and that lined up with what I suspected — the `!= 6` check was excluding Sundays from the increment branch for no good reason.

**Issue #4 (notifications):** The brief hinted the bug was architectural, so I compared `rate_song()` against the working `add_to_playlist()` myself and noticed `rate_song()` never called `create_notification()` at all. After I wrote the fix, it didn't actually work at first — turned out my Flask server was still running the old code since debug mode was off, so it hadn't reloaded. Took a bit to realize that was a server issue and not a bad fix.

**Issue #5 (playlist):** Found this one by just reading `get_playlist_songs()` line by line — the `songs[:-1]` slice at the end stood out immediately since it directly contradicted the function's own docstring saying it returns all songs.

**Where AI got it wrong:** When I was trying to reproduce Issue #4, Claude gave me curl commands with things like `<song_id>` and `<rater_user_id>` as placeholders, expecting me to swap in real values. I didn't catch that at first and just ran the commands as-is, which obviously failed since `<song_id>` isn't a real ID. It took a couple rounds of me pasting error output back before we sorted out that these were meant to be replaced, not typed literally. Small thing, but it cost some time and was a good reminder to double check example commands before running them instead of assuming they're copy-paste ready.

---

## Codebase Map

### Main files

- `app.py` — Flask app factory. Sets up the SQLAlchemy `db` instance and registers the 4 blueprints (`songs`, `playlists`, `users`, `feed`).
- `models.py` — all the SQLAlchemy models: `User`, `Song`, `Tag`, `Playlist`, `Rating`, `ListeningEvent`, `Notification`, plus association tables for friendships, song tags, and playlist entries. The playlist entries table has `position`, `added_by`, and `added_at` columns, so ordering and attribution are explicit, not just insertion order.
- `routes/songs.py`, `routes/playlists.py`, `routes/users.py`, `routes/feed.py` — these are all pretty thin. They parse the request, call a service function, and format the response.
- `services/streak_service.py` — listening streak logic.
- `services/feed_service.py` — the "friends listening now" and activity feeds.
- `services/search_service.py` — song search by title/artist.
- `services/notification_service.py` — creates and fetches notifications, but also owns `add_to_playlist` and `rate_song`, which is a little unexpected at first since you'd think those would live in playlist/song-specific services.
- `services/playlist_service.py` — playlist creation and retrieval.

### Pattern I noticed

Every route just parses input, calls a service, and handles the `ValueError` → status code mapping. All the actual logic lives in `services/`. Once I noticed this, it made debugging way faster — if something's wrong, the route is almost never the problem, so I could skip straight to the relevant service file.

The other thing I noticed: some service functions do more than one thing. `add_to_playlist()` both adds the song to the playlist AND creates a notification, all in one function. That's actually what made Issue #4 make sense once I found it — `rate_song()` follows the same "do the main thing" pattern but just never got the "also notify" part added.

### Data flow — adding a song to a playlist

1. `POST /playlists/<playlist_id>/songs` with `song_id` and `added_by` in the body.
2. `routes/playlists.py::add_song()` checks both fields exist, then calls `notification_service.add_to_playlist(...)`.
3. Inside that function: it looks up the song, the user adding it, and the playlist (404s if any are missing), adds the song to the playlist if it's not already there, and commits.
4. If the person adding the song isn't the original sharer, it calls `create_notification()` to notify whoever shared the song.
5. Route returns a 201 with a success message.

---

## Bug Fixes

### Issue #1: Listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/test_streaks.py -v` and `test_streak_increments_on_sunday` failed. The test has a user listen on Saturday (streak → 1) and then Sunday (should be streak → 2, since that's a consecutive day), but the actual streak stayed at 1.

**How I found the root cause:** Followed `record_listening_event()` into `update_listening_streak()` in `streak_service.py`. The branch that's supposed to increment the streak for a consecutive day was:
```python
elif days_since_last == 1 and today.weekday() != 6:
```
I wasn't sure offhand what `.weekday()` returns for different days, so I checked — Sunday is `6`. That means on a Sunday, `!= 6` is False, so even a genuinely consecutive listen gets kicked to the `else` branch and resets instead of incrementing.

**The root cause:** The increment condition had an extra, unnecessary check tied to the day of the week. There's nothing in the streak rules (per the function's own docstring) that says Sunday should behave differently — consecutive days should just increment, period. The `!= 6` check was excluding a valid case for no real reason.

**My fix:** Deleted `and today.weekday() != 6`, leaving just `days_since_last == 1`. Re-ran the streak tests — all 5 pass now, including the non-Sunday ones, so the other branches weren't affected.

---

### Issue #4: No notification when a friend rates my song

**How I reproduced it:** Rated a song (shared by one user) as a different user through `POST /songs/<song_id>/rate`. The rating itself saved fine. But checking the sharer's notifications afterward (`GET /users/<sharer_id>/notifications`) showed `count: 0` — nothing came through.

**How I found the root cause:** Since `add_to_playlist()` in the same file already does the "notify the sharer" thing correctly, I compared it directly to `rate_song()`. `add_to_playlist()` ends with a check for whether the person acting is the original sharer, and if not, calls `create_notification()`. `rate_song()` just saves/updates the `Rating`, commits, and returns — no notification call anywhere.

**The root cause:** It's not a broken condition or a typo — the notification step just isn't there. `rate_song()` was written to handle the rating itself but never got the equivalent of `add_to_playlist()`'s notify-the-sharer logic added on.

**My fix:** Added a block at the end of `rate_song()`, right after the commit, that checks if the rater isn't the sharer and calls `create_notification()` with a `song_rated` type — same pattern as the playlist one. Re-tested by rating the song again as a different user and confirmed the notification showed up with the right message. Also double-checked the rating itself still saves/updates properly, since I didn't want to break that in the process.

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Ran `pytest tests/test_playlists.py -v`. `test_playlist_returns_all_songs` failed — a playlist with 5 songs only returned 4 from `get_playlist_songs()`. The order test failed too, missing specifically the last song by position.

**How I found the root cause:** Read through `get_playlist_songs()` in `playlist_service.py`. The query looked fine — join, filter by playlist, order by position. But the return line was:
```python
return [song.to_dict() for song in songs[:-1]]
```
`songs[:-1]` drops the last item off the list. That's a plain Python slicing thing, not a query issue at all. It also directly contradicts the docstring, which says the function returns all songs in the playlist.

**The root cause:** The query itself was correct and returned every song in the right order — the bug was purely in how the results got sliced before being converted to dicts, silently dropping whatever song ended up last.

**My fix:** Changed `songs[:-1]` to just `songs`. Reran the playlist tests and all 3 pass now, including the empty-playlist case, so removing the slice didn't cause any issues when there's nothing to return.
