---
name: music163-export
description: >
  Export NetEase Cloud Music playlists to a local text file. Use when the user
  wants to export, backup, or save their playlists as text. Handles large
  playlists (1000+ songs), pagination, and browser data transfer limits.
disable-model-invocation: true
argument-hint: "[output-path]"
---

# NetEase Cloud Music Playlist Export

Export all playlists and songs from a logged-in NetEase Cloud Music account to a structured text file.

## Prerequisites

- User must be logged in at music.163.com (see `music163-playlist` skill for auth strategies)
- `chrome-devtools-mcp` must be available
- Page must have `window.asrsea` loaded (navigate to music.163.com first)

## Strategy: Browser-Side Aggregation

**Problem:** `evaluate_script` has a return value size limit (~50KB). Exporting 37 playlists / 4000 songs produces ~100KB of text. Naive approach (return all data in one call) fails.

**Solution:** Run the entire export pipeline inside the browser in phases:

1. **Phase 1 — Collect** (single evaluate_script): Fetch all playlist metadata + songs, store in `window.__exportData`
2. **Phase 2 — Format** (single evaluate_script): Generate the text content, store in `window.__musicTxt`
3. **Phase 3 — Transfer** (chunked reads): Extract text in <=40KB chunks via multiple evaluate_script calls
4. **Phase 4 — Write** (local): Concatenate chunks and write to disk

### Phase 1: Collect All Data

Execute this in `evaluate_script`. It handles:
- Fetching all playlists for the user
- Fetching tracks for each playlist (with 300ms rate limiting)
- Pagination for large playlists (>1000 tracks) via trackIds + song/detail fallback

```javascript
(async () => {
  const csrf = (document.cookie.match(/__csrf=([^;]+)/) || [])[1] || "";
  if (!csrf) return { error: "Not logged in" };
  if (typeof window.asrsea !== 'function') return { error: "asrsea not loaded" };

  const PK = "010001";
  const MOD = "00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7";
  const NONCE = "0CoJUm6Qyw8W8jud";

  async function post(path, data) {
    data.csrf_token = csrf;
    const enc = window.asrsea(JSON.stringify(data), PK, MOD, NONCE);
    const r = await fetch(`https://music.163.com/weapi${path}?csrf_token=${csrf}`, {
      method: "POST", headers: {"Content-Type": "application/x-www-form-urlencoded"},
      credentials: "include",
      body: `params=${encodeURIComponent(enc.encText)}&encSecKey=${enc.encSecKey}`
    });
    return r.json();
  }
  const delay = ms => new Promise(r => setTimeout(r, ms));

  // 1. Get uid
  const acctResp = await fetch("https://music.163.com/api/nuser/account/get", {
    method: "POST", credentials: "include",
    headers: {"Content-Type": "application/x-www-form-urlencoded"},
    body: "csrf_token=" + csrf
  });
  const acct = await acctResp.json();
  const uid = acct.account?.id;
  const nickname = acct.profile?.nickname || "Unknown";
  if (!uid) return { error: "Cannot get uid" };

  // 2. Get all playlists
  const plResp = await fetch(`https://music.163.com/api/user/playlist/?uid=${uid}&offset=0&limit=1000`, { credentials: "include" });
  const plData = await plResp.json();
  const playlists = plData.playlist || [];

  // 3. Fetch tracks for each playlist
  window.__exportData = {};
  window.__exportMeta = { uid, nickname };
  
  for (const pl of playlists) {
    const detail = await post("/v6/playlist/detail", { id: pl.id, n: 2000 });
    if (detail.code !== 200) { await delay(500); continue; }
    
    let tracks = (detail.playlist?.tracks || []).map(t => ({
      id: t.id, name: t.name, artist: (t.ar || []).map(a => a.name).join(" / ")
    }));
    
    // Handle pagination: if trackIds has more than tracks returned
    const allIds = (detail.playlist?.trackIds || []).map(t => t.id);
    if (tracks.length < allIds.length) {
      const missing = allIds.slice(tracks.length);
      for (let i = 0; i < missing.length; i += 500) {
        const batch = missing.slice(i, i + 500);
        const res = await post("/v3/song/detail", {
          c: JSON.stringify(batch.map(id => ({id}))),
          ids: JSON.stringify(batch)
        });
        if (res.songs) {
          tracks.push(...res.songs.map(t => ({
            id: t.id, name: t.name, artist: (t.ar || []).map(a => a.name).join(" / ")
          })));
        }
        await delay(300);
      }
    }
    
    window.__exportData[pl.id] = { name: pl.name, tracks, specialType: pl.specialType };
    await delay(300);
  }

  const totalTracks = Object.values(window.__exportData).reduce((s, p) => s + p.tracks.length, 0);
  return { status: "ok", playlists: playlists.length, totalTracks, uid, nickname };
})()
```

### Phase 2: Format Text

```javascript
(() => {
  const d = window.__exportData;
  const m = window.__exportMeta;
  const entries = Object.entries(d).sort((a, b) => {
    if (a[1].specialType === 5) return -1;
    if (b[1].specialType === 5) return 1;
    return a[1].name.localeCompare(b[1].name, 'zh-CN');
  });
  const total = Object.values(d).reduce((s, p) => s + p.tracks.length, 0);
  const lines = [
    `网易云音乐歌单导出`,
    `用户: ${m.nickname} (uid: ${m.uid})`,
    `导出时间: ${new Date().toLocaleString('zh-CN')}`,
    `歌单数: ${entries.length}`,
    `总歌曲数: ${total} (含重复)`
  ];
  for (const [, pl] of entries) {
    lines.push('', '========================================',
      `歌单: ${pl.name}`, `歌曲数: ${pl.tracks.length}`,
      '========================================');
    for (const t of pl.tracks) lines.push(`${t.artist} - ${t.name}`);
  }
  window.__musicTxt = lines.join('\n') + '\n';
  return { chars: window.__musicTxt.length, lines: lines.length };
})()
```

### Phase 3: Chunked Transfer

Read in chunks of 40000 chars:

```javascript
// Call N times, incrementing offset: 0, 40000, 80000, ...
(offset) => {
  const chunk = window.__musicTxt.substring(offset, offset + 40000);
  return { offset, length: chunk.length, done: offset + 40000 >= window.__musicTxt.length, chunk };
}
```

Pass `offset` as the argument. Stop when `done === true`.

### Phase 4: Write to Disk

Concatenate all chunks. Write via the `Write` tool or Python to the user's desired path.

## Output Format

```
网易云音乐歌单导出
用户: <nickname> (uid: <uid>)
导出时间: <datetime>
歌单数: <N>
总歌曲数: <N> (含重复)

========================================
歌单: <playlist_name>
歌曲数: <N>
========================================
<artist> - <song_name>
<artist> - <song_name>
...
```

## Rate Limits

| Operation | Interval |
|-----------|----------|
| Playlist detail | 300ms between calls |
| Song detail (batch) | 300ms, max 500 IDs per batch |
| Total export time | ~30s for 30 playlists, ~60s for 1500+ songs |

## Edge Cases

- **Subscribed playlists**: `n=2000` in playlist detail may only return 20 tracks. The trackIds array always has all IDs — use song/detail to fill gaps.
- **Deleted/taken-down songs**: song/detail may return fewer songs than IDs requested. These are silently skipped.
- **Empty playlists** (trackCount=0): Included in output with 0 songs, no tracks section.
- **Special playlists**: `specialType=5` = liked songs ("我喜欢的音乐"), `specialType=100` = private radar, `specialType=300` = AI generated.

## Boundaries

- Only exports the currently logged-in user's playlists
- Does not export play counts, comments, or creation dates per song
- Text format only (not CSV, JSON, or other structured formats unless user requests)
