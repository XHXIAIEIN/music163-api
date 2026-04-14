# API Reference

## weapi POST Endpoint Template

All write operations and most reads go through encrypted POST:

```
POST https://music.163.com/weapi/<path>?csrf_token=<csrf>
Content-Type: application/x-www-form-urlencoded
Cookie: MUSIC_U=<token>; __csrf=<csrf>
Referer: https://music.163.com
```

Request body (URL-encoded form data, NOT JSON):
```
params=<encrypted_base64>&encSecKey=<encrypted_hex>
```

The `params` field is the result of double-AES encrypting the JSON payload. The `encSecKey` is the RSA-encrypted random AES key. See SKILL.md "Encryption" section for implementation.

Every request's JSON payload should include `"csrf_token": "<csrf>"`.

---

## Encryption Constants

```
Nonce:    0CoJUm6Qyw8W8jud
IV:       0102030405060708
RSA Exp:  010001
RSA Mod:  00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725
          152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312
          ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424
          d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7
```

Process: `JSON.stringify(data)` → AES-CBC(nonce) → AES-CBC(randomKey) → base64 = `params`. Reverse randomKey → hex → RSA modpow → pad to 256 hex chars = `encSecKey`.

---

## Operation 0: Get Account Info

Get the currently logged-in user's uid and profile.

**POST** `/weapi/w/nuser/account/get`

Params:
```json
{ "csrf_token": "<csrf>" }
```

Response:
```json
{ "code": 200, "account": { "id": 12345678, "userName": "...", ... }, "profile": { "userId": 12345678, "nickname": "...", ... } }
```

Use `account.id` or `profile.userId` as the uid for subsequent calls.

---

## Operation 1: List User Playlists

**POST** `/weapi/user/playlist`

Params:
```json
{ "uid": "<uid>", "offset": 0, "limit": 30, "includeVideo": true, "csrf_token": "<csrf>" }
```

Response:
```json
{ "code": 200, "more": false, "playlist": [
  { "id": 123456, "name": "我喜欢的音乐", "trackCount": 500, "coverImgUrl": "...", "specialType": 5, "subscribed": false },
  { "id": 789012, "name": "My Playlist", "trackCount": 50, ... }
] }
```

- The first entry with `specialType: 5` is always the "liked songs" playlist ("我喜欢的音乐").
- `id` is the playlist ID used for all subsequent operations.
- Paginate with `offset` if user has many playlists. Set `limit: 1000` to get all at once.

---

## Operation 2: Get Playlist Songs

**POST** `/weapi/v6/playlist/detail`

Params:
```json
{ "id": "<playlist_id>", "n": 1000, "csrf_token": "<csrf>" }
```

Response:
```json
{ "code": 200, "playlist": {
  "id": 123456,
  "name": "My Playlist",
  "trackCount": 150,
  "tracks": [{ "id": 347230, "name": "Don't Know Why", "ar": [{"id": 34, "name": "Norah Jones"}], "al": {"id": 2846, "name": "Come Away with Me"} }, ...],
  "trackIds": [{"id": 347230}, {"id": 186016}, ...]
} }
```

- `tracks` contains full metadata but may be truncated at ~1000 songs.
- `trackIds` always contains all song IDs.
- For large playlists (>1000 tracks), use Operation 2b.

### Operation 2b: Get All Tracks (paginated)

**POST** `/weapi/playlist/track/all`

Params:
```json
{ "id": "<playlist_id>", "limit": 100, "offset": 0, "csrf_token": "<csrf>" }
```

Response:
```json
{ "code": 200, "songs": [{ "id": 347230, "name": "...", "ar": [...], "al": {...} }, ...] }
```

Paginate: increment `offset` by `limit` until songs returned < limit.

---

## Operation 3: Add Songs to Playlist

**POST** `/weapi/playlist/manipulate/tracks`

Params:
```json
{
  "op": "add",
  "pid": "<playlist_id>",
  "trackIds": "[347230, 186016, 29567]",
  "imme": "true",
  "csrf_token": "<csrf>"
}
```

**Important:** `trackIds` is a JSON-encoded string of an array, not a raw array.

Response:
```json
{ "code": 200, "status": 200, "message": "操作成功" }
```

Error codes:
- `502` — trackIds format error
- `401` — not authorized (not your playlist)
- `409` — song already exists in playlist (partial success possible)
- `200` — success

Batch strategy: add up to 200 songs per call. 500ms delay between batches.

---

## Operation 4: Remove Songs from Playlist

**POST** `/weapi/playlist/manipulate/tracks`

Params:
```json
{
  "op": "del",
  "pid": "<playlist_id>",
  "trackIds": "[347230, 186016]",
  "imme": "true",
  "csrf_token": "<csrf>"
}
```

Response: same structure as Operation 3.

---

## Operation 5: Create Playlist

**POST** `/weapi/playlist/create`

Params:
```json
{
  "name": "New Playlist Name",
  "privacy": 0,
  "ptype": 0,
  "csrf_token": "<csrf>"
}
```

- `privacy`: `0` = public, `10` = private
- `ptype`: `0` = normal, `"NORMAL"` also works

Response:
```json
{ "code": 200, "playlist": { "id": 9876543, "name": "New Playlist Name", ... } }
```

---

## Operation 6: Delete Playlist

**POST** `/weapi/playlist/remove`

Params:
```json
{ "ids": "[<playlist_id>]", "csrf_token": "<csrf>" }
```

**Important:** `ids` is a JSON-encoded array string (even for a single playlist). The endpoint is `/playlist/remove`, NOT `/playlist/delete` (which returns 400).

Response:
```json
{ "code": 200 }
```

Error: `code: 800` = playlist doesn't exist. Cannot delete the "我喜欢的音乐" playlist.

---

## Operation 7: Search Songs

**POST** `/weapi/search/get`

Params:
```json
{
  "s": "search query",
  "type": 1,
  "limit": 30,
  "offset": 0,
  "csrf_token": "<csrf>"
}
```

Search `type` values: `1` = songs, `10` = albums, `100` = artists, `1000` = playlists, `1002` = users, `1004` = MV, `1006` = lyrics.

Response (type=1):
```json
{ "code": 200, "result": { "songCount": 150, "songs": [
  { "id": 347230, "name": "海阔天空", "artists": [{"id": 11127, "name": "Beyond"}], "album": {"id": 2846, "name": "乐与怒"}, "duration": 326000 },
  ...
] } }
```

- `id` = song ID, `artists` = artists array (note: NOT `ar`), `album` = album object (note: NOT `al`), `duration` = duration in ms.
- **Warning:** `/weapi/search/get` response uses `artists`/`album`/`duration`, while `/weapi/v6/playlist/detail` uses `ar`/`al`/`dt`. Different field names for the same data.

**Alternative:** `/api/search/get` (POST, no encryption needed, same params and response).

**Avoid:** `/weapi/cloudsearch/get/web` — returns encrypted hex in the `result` field instead of plain JSON.

### Search Suggestions (lightweight)

**POST** `/weapi/search/suggest/web`

Params:
```json
{ "s": "keyword", "csrf_token": "<csrf>" }
```

Response: `{ "code": 200, "result": { "songs": [...], "artists": [...], "albums": [...] } }`

---

## Operation 8: Like / Unlike Song (Red Heart)

**POST** `/weapi/song/like`

Params:
```json
{ "trackId": "<song_id>", "like": true, "csrf_token": "<csrf>" }
```

- **`trackId`** (string, NOT `id` or `songId`) = the song's numeric ID as a string.
- `like: true` = add red heart, `like: false` = remove red heart.

Response:
```json
{ "code": 200 }
```

This is different from adding a song to the liked playlist via Operation 3 — this endpoint manages the red heart status specifically.

---

## Operation 9: Get Liked Song ID List

**POST** `/weapi/song/like/get`

Params:
```json
{ "uid": "<uid>", "csrf_token": "<csrf>" }
```

Response:
```json
{ "code": 200, "ids": [347230, 186016, 29567, ...], "checkPoint": 1697500000000 }
```

Returns an array of all liked song IDs (numeric). Use Operation 10 to get full metadata for specific songs.

---

## Operation 10: Get Song Details (by ID)

**POST** `/weapi/v3/song/detail`

Params:
```json
{
  "c": "[{\"id\":\"347230\"},{\"id\":\"186016\"}]",
  "ids": "[347230, 186016]",
  "csrf_token": "<csrf>"
}
```

`c` is a JSON-encoded array of `{id}` objects. `ids` is a JSON-encoded array of numbers.

Response:
```json
{ "code": 200, "songs": [
  { "id": 347230, "name": "Don't Know Why", "ar": [{"name": "Norah Jones"}], "al": {"name": "Come Away with Me"}, "dt": 186000 },
  ...
] }
```

Batch: up to ~1000 IDs per call.

---

## Operation 11: Bulk Text Import (Search + Add)

Composite workflow — given a list of song names, search each and add matches to a playlist:

1. Parse input: each line is `"artist - title"` or just `"title"`
2. Search each via Operation 7 (`limit: 1` for best match)
3. Collect song IDs, deduplicate
4. Batch add via Operation 3 (200 per batch, 500ms intervals)
5. Report: matched count, not-found list, added count, duplicate count

Optimization: unlike QQ Music, NetEase does NOT support batch search in a single call. Execute searches sequentially with 300ms delay.

---

## Response Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 301 | Not logged in / session expired |
| 302 | Need login |
| 400 | Bad request / invalid params |
| 404 | Resource not found (song taken down) |
| 409 | Song already in playlist |
| 502 | Encryption error / malformed params |
| 800 | Playlist not found |
| -462 | Anti-spam triggered (rate limited) |

---

## Full Workflow Example (Browser Context)

Complete flow in a single `evaluate_script` call. Uses `window.asrsea` (the page's built-in encryption):

```javascript
(async () => {
  // 1. Auth check
  const csrf = (document.cookie.match(/__csrf=([^;]+)/) || [])[1] || "";
  if (!csrf) return { error: "Not logged in: no __csrf cookie" };
  // MUSIC_U is HttpOnly — can't read it, but fetch sends it automatically

  // 2. Encryption helper (uses page's own function)
  if (typeof window.asrsea !== 'function') return { error: "window.asrsea not found — page not fully loaded" };

  const PUB_KEY = "010001";
  const MODULUS = "00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7";
  const NONCE = "0CoJUm6Qyw8W8jud";

  async function post(path, data) {
    data.csrf_token = csrf;
    const enc = window.asrsea(JSON.stringify(data), PUB_KEY, MODULUS, NONCE);
    const resp = await fetch(`https://music.163.com/weapi${path}?csrf_token=${csrf}`, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      credentials: "include",
      body: `params=${encodeURIComponent(enc.encText)}&encSecKey=${enc.encSecKey}`
    });
    return resp.json();
  }

  // 3. Get user info
  const acctResp = await fetch("https://music.163.com/api/nuser/account/get", {
    method: "POST", credentials: "include",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: "csrf_token=" + csrf
  });
  const acct = await acctResp.json();
  const uid = acct.account?.id;
  if (!uid) return { error: "Cannot get uid", detail: acct };

  // 4. Get playlists
  const playlists = await post("/user/playlist", { uid: String(uid), offset: 0, limit: 30, includeVideo: true });

  // 5. Example: add songs to a playlist
  const targetPid = playlists.playlist[1]?.id; // second playlist (first is "liked")
  const songIds = [/* array of song IDs */];
  const results = [];
  for (let i = 0; i < songIds.length; i += 200) {
    const batch = songIds.slice(i, i + 200);
    const res = await post("/playlist/manipulate/tracks", {
      op: "add", pid: String(targetPid),
      trackIds: JSON.stringify(batch), imme: "true"
    });
    results.push({ batch: Math.floor(i/200)+1, code: res.code });
    if (i + 200 < songIds.length) await new Promise(r => setTimeout(r, 500));
  }

  return { uid, playlistCount: playlists.playlist?.length, targetPid, batches: results };
})()
```
