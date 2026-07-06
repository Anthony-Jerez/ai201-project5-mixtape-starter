# Mixtape - Submission

## AI Usage

- Codebase orientation: I used AI to explore `routes/` and `services/` before touching any bug, read every route's endpoint and the service function it delegates to, and explain the data flow for song sharing and notifications. Also asked it to explain what `db.relationship` and the self referential `friendships` many to many table in `models.py` actually do at the ORM level, since I wasn't sure how that self join worked.
- Debugging: when `flask run` failed with `ModuleNotFoundError: No module named 'flask_sqlalchemy'`, had it diagnose why. It found that my shell's `flask` command was resolving to an unrelated Anaconda Python environment instead of the project's `.venv`, which already had the dependency installed correctly.
- Reproduction before fixing: for each bug, I had it actually run some of the existing tests or write small scripts to confirm the bug before touching any code
- Course correction on Issue 3: the working hypothesis, matching the assignment's own hint, was that `search_songs` produced duplicates because it was missing `.distinct()` on a join. Running the existing test suite showed all tests passing, which contradicted that hypothesis. Instead of accepting that first explanation, I had it dig further: comparing raw SQL row counts against the ORM's returned row counts, testing multiple multi tag songs matching at once, clearing the session identity map, and running the same query against the real seeded database. That confirmed SQLAlchemy's legacy `Query` API auto deduplicates entity results by primary key, so the hypothesized bug does not actually reproduce in this codebase. That finding is what led to swapping Issue 3 for Issue 1.

## Codebase Map

### Main Files and Roles

- `app.py`: Flask app factory (`create_app`). Owns the module level `db = SQLAlchemy()` instance, sets config from env vars with dev fallbacks (`DATABASE_URL`, `SECRET_KEY`), registers 4 blueprints (`songs`, `playlists`, `users`, `feed`) each under their own URL prefix, and calls `db.create_all()` inside an app context on every startup. There are no migrations.
- `models.py`: 7 models (`User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`) plus 3 raw association tables: `friendships` (symmetric self referential User to User), `song_tags` (Song to Tag), and `playlist_entries` (Playlist to Song, with extra `position`, `added_by`, and `added_at` columns). Because `Playlist.songs` is declared as a plain `secondary=` relationship, appending through it only touches `playlist_id` and `song_id`. It never populates `position` or `added_by`, even though those columns are required.
- `routes/`: one blueprint per resource (`songs.py`, `playlists.py`, `users.py`, `feed.py`). Every route parses the request, calls exactly one service function, and maps a raised `ValueError` to a 400 or 404 JSON response. The one exception is `routes/users.py`'s `get_user`, which queries `User` directly instead of going through a service.
- `services/`: all business logic and DB access lives here (`streak_service.py`, `feed_service.py`, `search_service.py`, `notification_service.py`, `playlist_service.py`). Every function takes primitive IDs, looks rows up with `db.session.get`, raises `ValueError("... not found")` on a miss, and commits directly. There is no repository layer and no rollback handling.
- `seed_data.py`: creates `Song` rows directly as model instances. This is the only place in the app that creates songs. There is no share song endpoint anywhere in `routes/`.
- `tests/`: one file per already documented bug (`test_streaks.py`, `test_search.py`, `test_playlists.py`), each seeding data directly through the session and asserting the behavior the service should produce.

### Data Flow: rating a song vs. adding a song to a playlist

`POST /songs/<song_id>/rate` (`routes/songs.py:29`) calls `notification_service.rate_song(user_id, song_id, score)`. `rate_song` (`services/notification_service.py:73`) validates the score, looks up the song and user, upserts a `Rating` row, commits, and returns it. It never calls `create_notification`, so the song's original sharer never hears about a new rating.

Compare that to `POST /playlists/<playlist_id>/songs` (`routes/playlists.py:43`), which calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)` (`services/notification_service.py:35`). This function follows the same validate then mutate then commit shape, but it additionally calls `create_notification(user_id=song.shared_by, ...)` (`services/notification_service.py:13`, the single shared constructor every `Notification` goes through) when the adder is not the original sharer.

Both flows follow the identical pattern up to the commit. Only one of them finishes with the `create_notification` call.

### Patterns Noticed

- Routes handle parsing, minimal presence checks, one service call, and error mapping. All logic and all `db.session` calls live in `services/`.
- Every service raises a bare `ValueError` for "not found" conditions, and every route catches it the same way. There is no custom exception hierarchy.
- Notification creation is centralized through `create_notification`, but only `add_to_playlist` calls it. Nothing else constructs a `Notification`, including `rate_song`.
- `db.create_all()` runs on every `create_app()` call, and songs only ever enter the system through `seed_data.py`.

## Issue Triage for Reference

| # | Title | Affected service |
|---|-------|-------------------|
| 1 | My listening streak keeps resetting | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` |
| 3 | The same song keeps showing up twice in search | `search_service.py` |
| 4 | I got notified when a friend added my song to a playlist but not when they rated it | `notification_service.py` |
| 5 | The last song in a playlist never shows up | `playlist_service.py` |

Root causes located while building the codebase map above:

- **Issue 4** (missing rating notification): `notification_service.py:73-110`. `rate_song` never calls `create_notification`, unlike the parallel `add_to_playlist` path described above.
- **Issue 5** (last playlist song missing): `playlist_service.py:66`. `get_playlist_songs` returns `songs[:-1]` instead of `songs`.
- **Issue 1** (streak resets): `streak_service.py`'s `update_listening_streak` has a suspicious `today.weekday() != 6` condition on the increment branch.

## Root Cause Analysis

### Issue 5: The last song in a playlist never shows up

**How I reproduced it:** Ran `pytest tests/test_playlists.py -v`. `test_playlist_returns_all_songs` failed with `assert 4 == 5`, and `test_playlist_returns_songs_in_order` failed because `"Track 5"` (the last song by position) was missing from the result. `test_empty_playlist_returns_empty_list` passed. Cross-checked against the real seeded database: all 3 seeded playlists have 7 songs at positions 1-7, and `get_playlist_songs` returned only 6 for each.

**How I found the root cause:** `GET /playlists/<id>/songs` to `routes/playlists.py` to `playlist_service.get_playlist_songs()`. `routes/playlists.py`'s `get_songs` route does nothing but call `get_playlist_songs` and return its result, so no slicing happens at the route level. Opening `playlist_service.py` and reading `get_playlist_songs` top to bottom, the query itself is correct (joins through `playlist_entries`, orders ascending by `position`). The moment of certainty was the return line sitting directly under a docstring `Note:` that says "This function returns all songs in the playlist" while the code right below it slices the list with `[:-1]` before returning, directly contradicting its own documented contract.

**The root cause:** `get_playlist_songs` (`services/playlist_service.py:66`) ends with `return [song.to_dict() for song in songs[:-1]]`. The query already returns the songs in the correct order. The `[:-1]` slice unconditionally drops the last element of that already correct list, regardless of playlist size, so the highest position song is always missing from the response.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs` so the full ordered list is returned. Verified with `pytest tests/test_playlists.py -v` — all 3 tests now pass. Also checked a boundary case the bug had been quietly hiding: a playlist with exactly 1 song. Under the old code, `[song][:-1]` evaluates to `[]`, so a 1-song playlist would have looked completely empty, not just short by one. Manually confirmed the fix now returns that 1 song correctly, and re-confirmed the 0-song case (`test_empty_playlist_returns_empty_list`) still passes.

### Issue 4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** No existing test covers this, so I wrote a one-off script (not saved to the repo) using an in-memory app context. Set up a `sharer` user, a `rater` user, one song owned by `sharer`, and a playlist owned by `sharer` with that song already in it (inserted directly into `playlist_entries`, matching how `seed_data.py` seeds playlists). Then:
1. Called `notification_service.rate_song(rater.id, song.id, 5)` and checked `get_notifications(sharer.id)` — result: `[]`, no notification created.
2. Called `notification_service.add_to_playlist(playlist.id, song.id, rater.id)` for the same song/users and checked again — result: one `song_added_to_playlist` notification appeared.

**How I found the root cause:** Followed the two call chains the README traces side by side: rating goes `routes/songs.py:rate` to `notification_service.rate_song`, adding to a playlist goes `routes/playlists.py:add_song` to `notification_service.add_to_playlist`. Both live in the same file, so I read them end to end back to back. `add_to_playlist` ends with a guard (`if song.shared_by != added_by_user_id`) followed by a `create_notification` call. `rate_song` ends at `db.session.commit(); return rating` with no such block anywhere. A repo-wide grep for `create_notification` confirmed there are only 2 references in the whole codebase: the function definition and that one call inside `add_to_playlist`. That ruled out the possibility that ratings notify through some other path.

**The root cause:** `rate_song` (`services/notification_service.py`, previously ending around line 110) validates the score, upserts the `Rating` row, and commits, but never calls `create_notification`. The parallel function `add_to_playlist` does call it, so the notification pipeline itself works fine. Ratings just never plug into it.

**My fix and side-effect check:** Added, right after the existing commit and before the return:
```python
    # Notify the person who originally shared the song (if it wasn't them who rated it)
    if song.shared_by != user_id:
        create_notification(
            user_id=song.shared_by,
            notification_type="song_rated",
            body=f"{rater.username} rated your song '{song.title}' {score}/5.",
        )
```
This mirrors `add_to_playlist`'s guard-then-notify shape exactly. `"song_rated"` is not a made up label either — it's the exact type string already named in `create_notification`'s own docstring example (`"e.g. 'song_added_to_playlist' or 'song_rated'"`), so this was clearly the intended type all along. Verified by re-running the reproduction script: `rate_song` now produces a `song_rated` notification for the sharer. Also checked the guard specifically by having the sharer rate their own song, and confirmed no notification is created, matching `add_to_playlist`'s self-add behavior. Ran the full test suite afterward to confirm `add_to_playlist`'s existing notification behavior is untouched.

### Issue 1: My listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/test_streaks.py -v`. `test_streak_increments_on_sunday` failed with `assert 1 == 2`. The test calls `update_listening_streak` for a user who listened on Saturday, then again on Sunday (a real consecutive day), and expects the streak to go from 1 to 2. Instead it reset to 1. The other 4 tests in the file (new user, consecutive non-Sunday day, same day repeat, skipped day) all passed, which pinned the failure specifically to the Sunday boundary.

**How I found the root cause:** Started at the documented call chain: `routes/songs.py`'s `listen` route calls `streak_service.record_listening_event`, which calls `update_listening_streak`. Read `update_listening_streak` against its own docstring, which states the rule plainly: "If the user listened yesterday: streak increments by 1." Comparing that sentence to the actual `elif` condition on the next few lines is what exposed the mismatch — the code checks `days_since_last == 1 and today.weekday() != 6`, and that second clause has no counterpart anywhere in the stated rules. Running the failing test with the Sunday date pinned down that `today.weekday() == 6` was exactly what flipped that condition to `False`.

**The root cause:** In `update_listening_streak`, the consecutive day increment branch is `elif days_since_last == 1 and today.weekday() != 6:`. Python's `datetime.weekday()` returns `6` for Sunday. So on any Sunday, even though `days_since_last == 1` correctly identifies a true consecutive day listen, the added `and today.weekday() != 6` clause makes the whole condition `False`, and execution falls through to the `else` branch, which resets the streak to 1 instead of incrementing it. The bug only ever shows up on the one day per week where a real streak continuation lands on a Sunday.

**My fix and side-effect check:** Changed `elif days_since_last == 1 and today.weekday() != 6:` to `elif days_since_last == 1:`, removing the erroneous clause entirely so the code matches its own documented rule. Verified with `pytest tests/test_streaks.py -v` — all 5 tests now pass, including `test_streak_increments_on_sunday`. Checked the other side of the boundary: consecutive non-Sunday days still increment correctly (`test_streak_increments_on_consecutive_day`, Monday to Tuesday, still passes). Also manually checked a skip-over-Sunday case (listened Friday, then Sunday, skipping Saturday): `days_since_last == 2`, which correctly still resets the streak to 1, confirming the fix didn't make Sundays increment unconditionally, only when the day is genuinely consecutive.