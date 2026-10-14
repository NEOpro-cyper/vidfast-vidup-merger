# Stream Resolver API — standalone (vidfast.vc + vidup.to)

Single-file Node.js server. No framework — just `server.js` + `package.json`.
The only dependency is `node-libcurl` (plain Node `fetch` gets 403'd by
vidfast.vc / vidup.to's Cloudflare TLS fingerprinting; libcurl passes).

## Run

```bash
npm install
node server.js            # listens on PORT env or 3000
```

## Endpoints

```
GET /api/movie/{tmdb-or-imdb-id}
GET /api/tv/{tmdb-or-imdb-id}/{season}/{episode}
```

Both accept TMDB ids (`27205`, `1399`) or IMDb ids (`tt15398776`, `tt0944947`).

### Query params

| Param    | Example            | Effect                                                                 |
|----------|--------------------|------------------------------------------------------------------------|
| `server` | `?server=2`, `?server=cobra` | Return one specific server (1-based index, or name, case-insensitive). Omit = race all servers, return the fastest healthy one. |
| `worker` | `?worker=1`        | Prefix the stream url with the Cloudflare worker proxy. Original stays in `originalUrl`. |
| `source` | `?source=vidup`    | Provider: `vidfast` (vidfast.vc, default) or `vidup` (vidup.to). Each has its own server list — vidfast: vRapid, vBlaze, Cobra, Cine, Bravo, Horizon; vidup: Euro, CineX, Zenith, Premier, Eclipse. |

Params combine: `?source=vidup&server=3&worker=1`

## Example

```
GET /api/tv/1399/1/1
```

```json
{
  "success": true,
  "type": "tv",
  "id": "1399",
  "season": 1,
  "episode": 1,
  "title": "Game of Thrones",
  "selectedBy": "fastest",
  "server": { "index": 1, "name": "vRapid", "streamMs": 535, "probeMs": 209, "totalMs": 744 },
  "url": "https://moon.zenoak.top/vd/.../master.m3u8",
  "originalUrl": "https://moon.zenoak.top/vd/.../master.m3u8",
  "worker": false,
  "stream": { "noReferrer": false, "url": "...", "tmdbId": 1399, "...": "full decrypted payload" },
  "servers": [ { "index": 1, "name": "vRapid", "ok": true, "totalMs": 744 }, ... ],
  "cached": false,
  "elapsedMs": 1832
}
```

Errors return `{ "success": false, "error": "..." }` with status 400 (bad params),
404 (no servers), or 502 (upstream failures).

## Behavior

- **Speed race**: all servers' stream endpoints are POSTed in parallel, decrypted,
  then each m3u8 is range-probed to measure real CDN latency. Fastest total wins.
- **Caching**: server list cached 15 min, decrypted streams 8 min (in-memory,
  per process, per source). Stale cache failures trigger one automatic refresh + retry.
- **Resilience**: handshake payloads are validated before the enc-dec GET requests
  (prevents opaque `414 Request-URI Too Large` when the site serves a Cloudflare
  challenge page) and transient upstream failures are retried once automatically.
- **CSRF**: enc-dec returns an empty csrf token for vidup — the `X-CSRF-Token`
  header is only sent when a token is present.
- **CORS**: `Access-Control-Allow-Origin: *` enabled for player/web use.
- **Debug**: run with `VIDFAST_DEBUG=1` to log page fetches and raw enc-dec
  stage-1 payloads to stderr.

## Env vars

| Var            | Default                                             |
|----------------|-----------------------------------------------------|
| `PORT`         | `3000`                                              |
| `WORKER_URL`   | `https://icy-art-1d5b-movy.ntb7nmytsd.workers.dev/` |
| `VIDFAST_DEBUG`| unset (set to `1` for request/handshake debug logs) |
