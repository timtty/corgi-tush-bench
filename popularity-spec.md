# Spec: popularity counts for the gallery sort

## Background

The gallery on `index.html` can now be sorted by Oldest, Newest or Most popular. Oldest and Newest use each run's `date` in `manifest.json` and need no server. Most popular ranks entries by **plays + votes**, so the server needs two new endpoints.

Until `/api/popularity` exists, the client ranks by votes alone, counting each run's `games` from `/api/leaderboard` for every prompt version. Plays are already being sent, and fail quietly until the server accepts them. Plays from before item 1 ships are lost.

This spec doesn't depend on any particular server stack. Map field and table names onto whatever the server already uses.

---

## 1. `POST /api/click`: record a play

The client sends this when someone clicks an entry's still to play it live, in the gallery or on a run's own page. It sends at most one per entry per page load.

- Body: `{ "id": "<run id>" }`
- Reject an `id` that isn't in `manifest.json` with `400`.
- Store `(run_id, ip_hash, created_at)`, using the same salted `ip_hash` the votes use.
- **Count each `(ip_hash, run_id)` at most once a day**, so refreshing the page doesn't add plays. An upsert on `(ip_hash, run_id, day)` is enough.
- Rate-limit to about 300 requests an hour per `ip_hash`. Over the limit, return `429`. The client ignores the response, so the message text doesn't matter.
- Response: `{ "ok": true }`. The client doesn't read it.

Plays from the vote view aren't sent, because those entries are anonymous there.

## 2. `GET /api/popularity`: counts per run

Response:

```json
{
  "runs": {
    "opus-5-20261012": { "clicks": 412, "votes": 87 },
    "gpt-6-20261016": { "clicks": 230, "votes": 91 }
  }
}
```

- `clicks`: deduplicated plays from item 1, all time.
- `votes`: the number of counted votes this run took part in, as either side, ties included, across all prompt versions. This is the same as wins + losses + ties on the leaderboard. If the dedup rule in `voting-defense-spec.md` item 1 ships, count deduplicated votes.
- Runs with no plays or votes can be left out. The client treats a missing run as 0 and 0.
- The client sorts by `clicks + votes`, highest first. On a tie, the newer entry goes first.
- The result can be cached for a minute or so.
- Any error, or a response that isn't JSON, makes the client fall back to votes from the leaderboard. If that fails too, it shows entries newest first.
