# IPFly JavaScript SDK

A dependency-free, high-performance JavaScript client for the [IPFly](https://ipfly.world) IP geolocation API. Drop it in with a `<script>` tag or `require`/`import` it — it handles caching, retries, rate limiting, request de-duplication, and batching for you.

- **Zero dependencies** — one file, works everywhere `fetch` exists
- **UMD build** — usable as a browser global (`window.IPFly`), CommonJS (`require`), or AMD
- **Smart caching** — TTL + LRU eviction, backed by memory, `localStorage`, or `sessionStorage`
- **Resilient** — automatic retry with exponential backoff + jitter on network/5xx failures
- **Efficient** — in-flight request de-duplication and a client-side rate limiter
- **Batch-friendly** — look up many IPs at once with bounded concurrency
- **Observable** — lifecycle hooks for requests, responses, errors, and cache hits

---

## ⚠️ Before you deploy: a note on your API token

IPFly's own documentation is explicit about this, and it's worth repeating here: **any token embedded in client-side JavaScript is visible to anyone who opens dev tools or views page source.** This SDK is built to run in the browser (that's what makes it useful for "drop it on your site" use cases), but for a production public website you have two safe options:

1. **Proxy it (recommended).** Run a thin backend endpoint that holds the real token, and point this SDK's `baseURL` at *your own* endpoint instead of `https://ipfly.world/api`. Your backend forwards the request to IPFly and returns the result. The token never reaches the browser.
2. **Restrict the token's origin.** In your IPFly dashboard, lock the token to your site's specific domain(s). A copied/stolen token then simply won't work from anywhere else. This reduces risk but doesn't eliminate it — treat it as a mitigation, not a substitute for proxying.

If you're prototyping, calling IPFly directly from the browser with a domain-restricted token is a reasonable tradeoff. If you're shipping to production with meaningful traffic, proxy it.

---

## Installation

### Option A — Script tag from our public CDN with versioning (no build step)

```html
<script src="https://ipfly.world/libs/javascript-sdk/1.0.0.js"></script>
<script>
  const ipfly = IPFly.createClient({ token: 'YOUR_TOKEN' });
</script>
```

### Option B — CommonJS / bundlers (Webpack, Rollup, esbuild, Node)

```js
const IPFly = require('./ipfly-sdk.js');
const ipfly = IPFly.createClient({ token: 'YOUR_TOKEN' });
```

### Option C — ES modules

Most bundlers will happily import the UMD file directly:

```js
import IPFly from './ipfly-sdk.js';
const ipfly = IPFly.createClient({ token: 'YOUR_TOKEN' });
```

No npm package is published — host `ipfly-sdk.js` yourself (e.g. alongside your other static assets, or on your own CDN) and reference it by URL.

---

## Quick start

```js
const ipfly = IPFly.createClient({
  token: 'YOUR_TOKEN',
  include: 'security' // optional: request the security/ASN field set on every call
});

// Look up a specific IP
ipfly.lookup('8.8.8.8').then((data) => {
  console.log(data.city, data.country_name, data.security.is_vpn);
});

// Look up the caller's own IP (omit the ip argument)
ipfly.lookupSelf().then((data) => {
  console.log('You appear to be in', data.city);
});

// async/await works the same way
async function whoIsThis(ip) {
  try {
    const data = await ipfly.lookup(ip);
    return data;
  } catch (err) {
    console.error(err.code, err.message);
  }
}
```

---

## Configuration reference

All options are passed to `IPFly.createClient({ ... })`.

| Option | Type | Default | Description |
|---|---|---|---|
| `token` | `string` | **required** | Your IPFly API token. |
| `baseURL` | `string` | `https://ipfly.world/api` | Override to point at your own backend proxy. |
| `include` | `string` | `null` | Default `include` value sent with every request (e.g. `"security"`). Can be overridden per call. |
| `timeout` | `number` | `8000` | Per-request timeout in ms before the request is aborted. |
| `retries` | `number` | `2` | Retry attempts for network errors and `5xx` responses. `4xx` errors are never retried. |
| `retryDelay` | `number` | `300` | Base delay (ms) for exponential backoff between retries (jitter is added automatically). |
| `rateLimit` | `number` | `0` (unlimited) | Max requests/second this client will *issue*. Useful to stay comfortably under your plan's limits. |
| `concurrency` | `number` | `6` | Default max parallel requests for `batchLookup`. |
| `cache` | `object` | see below | Response cache settings. |
| `cache.enabled` | `boolean` | `true` | Set `false` to disable caching entirely. |
| `cache.ttl` | `number` | `300000` (5 min) | How long a cached response stays valid, in ms. |
| `cache.maxSize` | `number` | `500` | Max cached entries before least-recently-used entries are evicted. |
| `cache.storage` | `'memory' \| 'localStorage' \| 'sessionStorage'` | `'memory'` | Where cached responses are stored. Web Storage persists across page loads. |
| `fetch` | `function` | global `fetch` | Supply your own `fetch` implementation (e.g. `node-fetch`) for non-browser environments. |
| `debug` | `boolean` | `false` | Log internal lifecycle events to the console (tokens are redacted in logs). |
| `onRequest` | `(ip, url) => void` | `null` | Hook fired right before a network request is sent. |
| `onResponse` | `(ip, data, fromCache) => void` | `null` | Hook fired after a successful lookup. |
| `onError` | `(ip, error) => void` | `null` | Hook fired when a lookup ultimately fails (after retries). |
| `onCacheHit` | `(ip, data) => void` | `null` | Hook fired when a cached response is served instead of a network call. |

---

## API

### `IPFly.createClient(config)`
Creates a new client instance. Equivalent to `new IPFly.Client(config)`.

### `client.lookup(ip?, options?)`
Look up a single IP. Omit `ip` (or pass `null`/`undefined`) to geolocate the caller.

```js
client.lookup('1.1.1.1', {
  include: 'security',   // overrides the client-level `include` for this call only
  skipCache: true,        // force a fresh network request, bypassing the cache
  signal: abortController.signal // optional AbortSignal to cancel this specific call
});
```
Returns a `Promise` that resolves with the parsed JSON response (see [Response shape](#response-shape)) or rejects with an `IPFlyError`.

### `client.lookupSelf(options?)`
Shorthand for `client.lookup(null, options)` — geolocates the visitor's own IP.

### `client.batchLookup(ips, options?)`
Looks up an array of IPs with bounded concurrency (`options.concurrency` overrides the client default). This call **never rejects** — each entry in the resolved array is either a success or a failure record, so one bad IP doesn't stop the batch:

```js
const results = await client.batchLookup(['8.8.8.8', '1.1.1.1', 'not-an-ip']);
// [
//   { ip: '8.8.8.8', ok: true,  data: { ... } },
//   { ip: '1.1.1.1', ok: true,  data: { ... } },
//   { ip: 'not-an-ip', ok: false, error: IPFlyError }
// ]
```

### `client.clearCache()`
Empties the response cache immediately.

### `client.cacheStats()`
Returns `{ enabled, size, ttl, maxSize }` — useful for debugging or an admin/status panel.

### `client.destroy()`
Clears the cache and any in-flight request bookkeeping. Call this if you're tearing down a client instance (e.g. in a single-page app route change).

### `IPFly.isValidIP(ip)`
Standalone helper — validates an IPv4 or IPv6 string without making a request.

---

## Response shape

Successful lookups resolve with the same JSON structure IPFly's API returns, unchanged — see the [IPFly API docs](https://ipfly.world) for the full field reference. Shape depends on your plan and the `include` parameter you request (e.g. `security` and ASN/company fields require Pro or higher).

```json
{
  "ip": "8.8.8.8",
  "hostname": "dns.google",
  "country_name": "United States",
  "city": "Mountain View",
  "latitude": "37.4056",
  "longitude": "-122.0775",
  "time_zone": { "name": "America/New_York", "...": "..." },
  "asn": { "asn": "AS15169", "name": "Google LLC", "...": "..." },
  "security": { "is_vpn": false, "is_tor": false, "...": "..." }
}
```

---

## Error handling

All failures reject with an `IPFlyError` (a standard `Error` subclass) carrying extra fields:

| Field | Description |
|---|---|
| `message` | Human-readable description. |
| `status` | HTTP status code, if the request reached the server (`null` for network/timeout errors). |
| `code` | Machine-readable code — see table below. |
| `cause` | The underlying error or response body, when available. |

| `code` | Meaning |
|---|---|
| `MISSING_TOKEN` | No token was supplied when creating the client. |
| `NO_FETCH` | No `fetch` implementation is available in this environment; supply one via `config.fetch`. |
| `INVALID_IP` | The IP string failed local validation before any request was sent. |
| `TIMEOUT` | The request exceeded `timeout` or was aborted. |
| `NETWORK_ERROR` | The request failed before a response was received (offline, DNS, CORS, etc). |
| `HTTP_ERROR` / server-provided code | The API responded with a non-2xx status. Check `status` (401 = bad token, 403 = plan/permission issue, 404 = private/bogon IP, 429/5xx = retry-worthy). |
| `BAD_RESPONSE` | The server responded but the body wasn't valid JSON. |

```js
try {
  await client.lookup('8.8.8.8');
} catch (err) {
  if (err.code === 'TIMEOUT') {
    // retry later, show a "slow connection" message, etc.
  } else if (err.status === 401) {
    // token is invalid — surface a config error, don't retry
  }
  console.error(err.message);
}
```

---

## Performance notes

- **Caching** avoids re-querying the same IP within the TTL window — handy for pages that look up the same visitor repeatedly, or dashboards re-rendering a list of known IPs.
- **De-duplication**: if your UI fires `lookup('1.2.3.4')` from three components in the same tick, only one network request goes out; all three get the same result.
- **Rate limiting** is a courtesy limiter on the *client* side — it smooths bursts (e.g. rendering a table of 200 rows) so you don't blow through your plan's requests/second in a single frame. It does not talk to IPFly's servers to discover your actual plan limits.
- **Retries** only apply to transient failures (network errors, timeouts, `5xx`). A `401`/`403`/`404` fails fast since retrying won't fix a bad token or a private IP.

---

## Example: rendering a "Welcome from {city}" banner

```html
<script src="/vendor/ipfly-sdk.js"></script>
<script>
  const ipfly = IPFly.createClient({
    token: 'YOUR_TOKEN',
    cache: { storage: 'sessionStorage', ttl: 30 * 60 * 1000 }
  });

  ipfly.lookupSelf()
    .then((data) => {
      document.getElementById('banner').textContent =
        `Welcome from ${data.city || data.country_name}!`;
    })
    .catch(() => {
      document.getElementById('banner').textContent = 'Welcome!';
    });
</script>
```

## Example: proxying instead of exposing your token

```js
// Browser: point at your own backend, no token here at all
const ipfly = IPFly.createClient({
  token: 'unused-placeholder', // your backend holds the real token
  baseURL: '/api/geo'          // your server-side route
});
```

```js
// Your backend (e.g. Express), holding the real token server-side
app.get('/api/geo', async (req, res) => {
  const ip = req.query.ip || req.ip;
  const upstream = await fetch(
    `https://ipfly.world/api?token=${process.env.IPFLY_TOKEN}&ip=${ip}&include=security`
  );
  res.status(upstream.status).json(await upstream.json());
});
```

---

## Browser support

Any environment with `Promise`, `Map`, and `fetch` (or a supplied `fetch` polyfill/implementation) works — all evergreen browsers, and Node 18+ for server-side use. `AbortController` is used opportunistically for timeouts/cancellation and is optional (its absence just disables timeout support, it won't throw).

## License

MIT — use it, fork it, ship it.
