# Mixtape submission

AI usage

I used AI as a navigation and debugging partner while working through the codebase. I asked it to explain how SQLAlchemy models map to database tables, how foreign keys and association tables connect users, songs, tags, playlists, and listening events, and how to follow a request from a route to a service and then to the database.

For debugging, I used the AI to help identify which service owned each issue and to suggest focused checks. I used seed_data.py to create known local data, queried the SQLite database to find generated user, song, playlist, and event IDs, and used curl requests to reproduce API behavior. I then compared the API responses with the database records and read the relevant route, service, and model code myself.

The AI helped me understand concepts such as rolling time windows versus calendar-day boundaries, descending query order, deduplication with a set, SQL joins, and why a list slice removes the last item. It also helped me format JSON responses so I could read the results more easily.

I verified the suggestions by checking the actual code, running the API requests, inspecting database values, reviewing the resulting diffs, and running the available tests. Some early explanations pointed me toward the wrong service or made the feed and streak issues seem related, so I corrected course by tracing each endpoint separately. I also checked that removing the search join did not remove tags, because tags were already loaded through the Song relationship.

# Mixtape codebase map

app.py creates the Flask app, connects the database, registers the routes, and creates the database tables.

models.py defines the database tables and their relationships. The main models are User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification.

The association tables are:

- friendships connects users to other users.
- song_tags connects songs to tags.
- playlist_entries connects playlists to songs. It also stores the song position, who added it, and when it was added.

The models use UUIDs for IDs. The to_dict methods turn database objects into dictionaries for JSON responses.

The routes folder handles HTTP requests:

- routes/songs.py handles searching, viewing, rating, and listening to songs.
- routes/playlists.py handles creating playlists and viewing or adding playlist songs.
- routes/users.py handles user profiles, streaks, and notifications.
- routes/feed.py handles the friends listening feed and activity feed.

The services folder contains the main application logic:

- services/search_service.py searches for songs.
- services/streak_service.py records listening events and updates streaks.
- services/feed_service.py builds the friends listening and activity feeds.
- services/playlist_service.py creates and retrieves playlists and their songs.
- services/notification_service.py creates notifications and saves ratings.

Example data flow for rating a song:

1. A client sends POST /songs/<song_id>/rate with a user_id and score.
2. routes/songs.py receives the request and checks the input.
3. It calls rate_song in services/notification_service.py.
4. rate_song checks the score, finds the user and song, and looks for an existing rating.
5. It updates the existing rating or creates a new one.
6. It saves the rating in the database.
7. If another user rated the song, it creates a notification for the person who shared the song.
8. The route returns the rating as JSON.

The general flow is:

HTTP request -> route/controller -> service -> model/database -> JSON response

seed_data.py creates local test data. The tests folder contains tests for streaks, searches, and playlists. requirements.txt lists the required Python packages. README.md contains setup and project information.

The main pattern is that routes handle requests and responses, services handle application logic, and models handle database data and relationships.

Regression test

I added test_playlist_includes_final_song in tests/test_playlists.py. It checks that the final song, Track 5, is included in the playlist response. This test would fail with the original songs[:-1] behavior and passes after returning all songs.

Root cause analysis

**Issue #1 - My listening streak keeps resetting**

How I reproduced it: I used the seeded users and their listening data, then traced the streak endpoint from routes/users.py to services/streak_service.py. I compared a user's current streak before and after recording a listening event with curl. The failure happened when the previous listening date was the day before and the current day was Sunday.

How I found the root cause: I started at the GET /users/<user_id>/streak route and followed it to get_streak() and update_listening_streak(). I checked the User fields listening_streak and last_listened_at in the database. The specific cause was in the condition that handled a one-day gap.

The root cause: update_listening_streak() treated Sunday as an exception. Even when today was exactly one calendar day after the last listening date, the condition was false on Sunday, so the code went to the reset branch and changed the streak to 1.

The fix: I removed the Sunday-specific exclusion from the consecutive-day condition. A listening event from yesterday now increments the streak regardless of which weekday today is.

Side-effect check: I checked the same-day behavior, where listening again on the same day should not increase the streak, and the skipped-day behavior, where the streak should reset to 1. I also checked that the listening event is still created and saved.

**Issue #2 - Friends Listening Now shows people from yesterday**

How I reproduced it: I seeded the database, found the user IDs, and called GET /feed/<user_id>/listening-now with curl. I compared the returned listened_at values with the seeded listening events. An event from late the previous night was still included the next morning.

How I found the root cause: I followed routes/feed.py to get_friends_listening_now() in services/feed_service.py. I inspected the ListeningEvent table and checked the user_id and listened_at values. I also checked the query ordering and the duplicate-removal loop. The specific mismatch was in the definition of the recent time boundary.

The root cause: the feed used a rolling 24-hour cutoff. An event from 11 PM yesterday is less than 24 hours old at 9 AM today, so it passed the filter even though the feature requires listening from today's calendar date.

The fix: I changed the cutoff to today's UTC midnight. The query still filters on ListeningEvent.listened_at and orders events from newest to oldest, but events from the previous calendar day are no longer included.

Side-effect check: I checked that events from today still appear, events from yesterday do not appear, and the feed still returns only the most recent event for each friend.

**Issue #3 - The same song keeps showing up multiple times in search**

How I reproduced it: I seeded the database and called GET /songs/search?q=Anthem with curl. The response returned the same Crown Heights Anthem song three times. Each copy had the same song ID and the same tags.

How I found the root cause: I followed routes/songs.py to search_songs() in services/search_service.py. I checked the Song, Tag, and song_tags data in the database. The search query used an outer join from Song to song_tags, and the song had three tags. That produced three query rows for one song before the results were converted to dictionaries.

The root cause: the search query joined each song to every matching tag row, so a song with three tags appeared three times in the raw result. The to_dict() method then converted each duplicate Song object into the same response dictionary.

The fix: I removed the unnecessary outer join. The Song.tags relationship already supplies the song's tags when to_dict() is called, so the search query can return each matching Song once while keeping all tag names in one tags list.

Side-effect check: I searched for a song with several tags, a song with one tag, and a song with no tags. I checked that each song appears once and that its tags are still returned together.

**Issue #4 - Rating a song did not notify the song owner**

How I reproduced it: I seeded the database, selected a song and two users, and sent a POST request to /songs/<song_id>/rate with the second user's ID and a score. I then requested the original song sharer's notifications. The rating was saved, but there was no rating notification.

How I found the root cause: I followed routes/songs.py to rate_song() in services/notification_service.py. I checked the Song.shared_by field, the rater User record, the Rating record, and the Notification model. The rating save path committed the Rating but did not create a notification afterward.

The root cause: rate_song() handled validation and creation or updating of the Rating, but it had no notification step for the user who originally shared the song. The playlist notification logic existed separately in add_to_playlist().

The fix: I added a notification after saving the rating. When the rater is not the song owner, the code sends a notification to song.shared_by with notification type song_rated and a message containing the rater, song, and score.

Side-effect check: I checked that a rating from another user creates a notification, that the song owner does not receive a notification for rating their own song, that invalid scores still return an error, and that an existing rating can still be updated.

**Issue #5 - The last song in a playlist never shows up**

How I reproduced it: I seeded the database and requested a playlist's songs through GET /playlists/<playlist_id>/songs. I compared the returned list with the playlist_entries rows and saw that the final ordered song was missing.

How I found the root cause: I followed routes/playlists.py to get_playlist_songs() in services/playlist_service.py. I checked the playlist_entries rows, the position values, and the query ordering. The query returned the complete ordered list, but the return statement changed it before sending the response.

The root cause: get_playlist_songs() used songs[:-1] when converting the query results to dictionaries. Python slicing with [:-1] removes the last item, so the final playlist song was always excluded from the response.

The fix: I changed the return statement to convert songs directly with [song.to_dict() for song in songs]. This keeps the ascending playlist order and includes every song.

Side-effect check: I checked that the playlist still follows the position order, that the first and middle songs remain present, and that the final song is now included.
