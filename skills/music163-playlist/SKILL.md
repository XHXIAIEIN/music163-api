---
name: music163-playlist
description: >
  Automate NetEase Cloud Music (网易云音乐) playlist management via reverse-engineered
  internal Web APIs. Use when the user wants to list playlists, add/remove songs,
  search tracks, create/delete playlists, manage favorites (red heart), or
  bulk-import songs from text into NetEase Cloud Music.
---

# NetEase Cloud Music Playlist Automation

Manage NetEase Cloud Music playlists through internal Web APIs reverse-engineered from music.163.com.

All write APIs use the `/weapi/` encrypted endpoint. Reads can use either `/weapi/` or legacy `/api/`. Auth requires `MUSIC_U` cookie (login credential) and `__csrf` token.

## Authentication

Try in priority order, stop on first success:

**Strategy A — Auto-extract (when chrome-devtools-mcp is available)**

Check if MCP tools `list_pages`, `select_page`, `evaluate_script` exist. If so:

1. `list_pages` → find page with URL containing `music.163.com`
2. If none → `new_page` to open `https://music.163.com`
3. `select_page` to select it
4. `evaluate_script`:

```javascript
(async () => {
  const csrf = (document.cookie.match(/__csrf=([^;]+)/) || [])[1] || "";
  if (!csrf) return { error: "Not logged in: no __csrf cookie. Please log in at music.163.com first." };
  // MUSIC_U is HttpOnly — JS can't read it, but fetch sends it automatically.
  // Verify login by calling account API:
  const resp = await fetch("https://music.163.com/api/nuser/account/get", {
    method: "POST", credentials: "include",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: "csrf_token=" + csrf
  });
  const data = await resp.json();
  if (data.code !== 200 || !data.account) return { error: "Not logged in or session expired. Please log in at music.163.com." };
  return { csrf, uid: data.account.id, nickname: data.profile?.nickname, status: "ok" };
})()
```

Note: `MUSIC_U` is `HttpOnly` — JavaScript cannot read it directly, but browser `fetch` with `credentials: 'include'` sends it automatically. Auth verification must be done by calling an API, not by reading cookies.

**Strategy B — File / env vars**

Check in order:
- Env vars `MUSIC_U` and `NCM_UID`
- `.env` file in project root (`MUSIC_U=xxx`, `NCM_UID=xxx`)

**Strategy C — Ask user (last resort)**

Tell the user: open music.163.com → F12 → Application → Cookies → copy `MUSIC_U` value.

## Encryption (weapi)

All `/weapi/` requests require encrypted POST body.

**Strategy A — Use page's built-in function (preferred):**

music.163.com exposes `window.asrsea(text, pubKey, modulus, nonce)` which returns `{encText, encSecKey}`. This is the page's own encryption function — no need to reimplement.

```javascript
function encrypt(data) {
  data.csrf_token = csrf;
  const enc = window.asrsea(JSON.stringify(data),
    "010001",
    "00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7",
    "0CoJUm6Qyw8W8jud");
  return `params=${encodeURIComponent(enc.encText)}&encSecKey=${enc.encSecKey}`;
}
```

Check availability: `typeof window.asrsea === 'function'`. If unavailable, page may not be fully loaded — wait or reload.

**Strategy B — Manual CryptoJS (fallback for non-browser or if asrsea unavailable):**

See reference.md "Encryption Constants" section for the full manual implementation using CryptoJS AES + BigInt RSA.

## How to Call APIs

**With browser (Strategy A):** Execute encryption + `fetch` via `evaluate_script` in page context. All fetch calls must include `credentials: 'include'`.

**Without browser (Strategy B/C):** Use Python with `pycryptodome` for AES encryption + built-in `pow()` for RSA, then `urllib`/`requests` POST with headers:
```
Cookie: MUSIC_U=<MUSIC_U>
Referer: https://music.163.com/
Content-Type: application/x-www-form-urlencoded
```

Note: weapi POST body is `application/x-www-form-urlencoded` (NOT JSON). Send as `params=<encrypted>&encSecKey=<key>`.

## Operations

| # | Operation | Endpoint | Type |
|---|-----------|----------|------|
| 0 | Get account info (uid) | `/weapi/w/nuser/account/get` | Read |
| 1 | List user playlists | `/weapi/user/playlist` | Read |
| 2 | Get playlist songs | `/weapi/v6/playlist/detail` | Read |
| 3 | Add songs to playlist | `/weapi/playlist/manipulate/tracks` | Write |
| 4 | Remove songs from playlist | `/weapi/playlist/manipulate/tracks` | Write |
| 5 | Create playlist | `/weapi/playlist/create` | Write |
| 6 | Delete playlist | `/weapi/playlist/remove` | Write |
| 7 | Search songs | `/weapi/search/get` | Read |
| 8 | Like / unlike song (red heart) | `/weapi/song/like` | Write |
| 9 | Get liked song ID list | `/weapi/song/like/get` | Read |
| 10 | Get song details (by ID) | `/weapi/v3/song/detail` | Read |
| 11 | Bulk text import (search + add) | Composite workflow | Write |

For full API specs, params, and code examples for each operation, read [reference.md](reference.md).

## Key Facts

| Topic | Detail |
|-------|--------|
| `MUSIC_U` vs `__csrf` | `MUSIC_U` = login session token (long-lived, **HttpOnly** — JS cannot read it). `__csrf` = CSRF token (readable from cookie). Both sent automatically by browser fetch with `credentials: 'include'`. Verify login by calling `/api/nuser/account/get`, not by checking cookies. |
| Encryption | All `/weapi/` endpoints require double-AES + RSA encrypted body. **Prefer `window.asrsea(text, pubKey, modulus, nonce)`** — the page's own encryption function. Falls back to manual CryptoJS implementation. |
| Content-Type | weapi uses `application/x-www-form-urlencoded`, NOT JSON. Body: `params=<enc>&encSecKey=<key>`. |
| Favorites playlist | "My Liked Music" (我喜欢的音乐) is the user's first playlist (index 0 in playlist list). Unlike QQ Music, there is NO fixed dirId — get it from the user's playlist list. |
| Like (red heart) vs playlist add | Red heart = `/weapi/song/like` with `trackId` (string) + `like` (bool). Adding to favorites playlist = `/weapi/playlist/manipulate/tracks` with the favorites playlist ID. These are DIFFERENT operations. |
| Song IDs | NetEase uses numeric `id` (e.g. `347230`). `trackIds` param takes JSON array of IDs: `[347230, 186016]`. |
| `window.asrsea` | The page exposes `window.asrsea(text, pubKey, modulus, nonce)` → `{encText, encSecKey}`. Use this instead of manual encryption when in browser context. Check `typeof window.asrsea === 'function'` first. |
| `cloudsearch` encrypted response | `/weapi/cloudsearch/get/web` returns encrypted hex in `result` field. Use `/weapi/search/get` instead — returns plain JSON. |
| Delete playlist | Use `/weapi/playlist/remove` with `{ids: "[pid]"}`, NOT `/weapi/playlist/delete`. |
| `chrome-devtools-mcp` | Required for browser automation. `claude-in-chrome` may block music.163.com. |
| CryptoJS fallback | Only needed if `window.asrsea` is unavailable. Inject via CDN: `crypto-js/4.2.0/crypto-js.min.js`. |

## Rate Limits

| Operation | Concurrency | Interval |
|-----------|-------------|----------|
| Read (playlist, search) | 5 parallel | 300ms |
| Write (add/remove tracks) | Sequential | 500ms per batch |
| Search | Sequential | 300ms per query |
| Like/unlike | Sequential | 300ms per call |

## Output Format

After each operation, report:
- **Operation**: what was done
- **Summary**: success / existed / failed counts
- **Key data**: playlist ID, song count, batch count
- **Errors**: specific error details if any

## Quality Bar

- Read current state before writes (list → add → list to verify).
- On mid-batch errors, log succeeded vs failed batches. Do not retry the entire operation.
- Auth errors (code `301` or `-462`) → re-extract MUSIC_U, retry once.
- Network errors → surface to user, let them decide.
- Before adding songs, check if they already exist in the target playlist to avoid duplicates.

## Boundaries

- Search returns first match; ambiguous song names may mismatch. Confirm with user when in doubt.
- Only operates on the **currently logged-in user's** playlists. Cannot modify others' playlists.
- Unofficial API — may break if NetEase changes backend.
- Unlike QQ Music, NetEase has stricter rate limiting. Respect intervals to avoid IP blocks.
