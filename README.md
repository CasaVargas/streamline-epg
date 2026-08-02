# streamline-epg

EPG source manifest for [Streamline](https://github.com/CasaVargas/Streamline), served as a
static file at **https://epg.getstreamline.tv/manifest.json**.

The app fetches this at most once every 24 h, and falls back to a copy bundled in the app
binary if it is unreachable — so a bad deploy here degrades to "no automatic guide", never
to a broken one. The fetch is capped at 8 s per request / 12 s per resource, so an
unresolvable host costs milliseconds, not a stalled launch.

## Why this exists

Streamline picks EPG feeds for a lineup with no user configuration. Keeping the registry
outside the app means a feed going dark can be routed around without shipping an update.

## Schema

Decoded by `EPGSourceManifest` / `EPGSource`. Every field below is required.

| field | notes |
|---|---|
| `id` | stable identifier, appears in logs |
| `url` | the XMLTV feed (`.xml` or `.xml.gz`) |
| `regions` | lowercase 2-letter codes, matched against the lineup fingerprint |
| `kind` | `national` / `locals` / `sports` / `provider` — selection precedence, lower wins |
| `idConvention` | `iptv-org` / `dotted-name` / `opaque` — see below |
| `forwardSpanDays` | how far ahead the feed publishes |
| `approxChannels` | ties break toward more channels, then smaller size |
| `sizeMB` | download size |
| `health` | `ok` / `degraded` / `dead` — `dead` is retained but never selected |

### `idConvention` is not cosmetic

A feed is only selected when its convention matches the lineup's. Measured on a real
136-channel lineup using iptv-org ids (`cnn.us`), the region-correct but dotted-name
epgshare01 feed (`NBC.Sports.Boston.HD.us2`) matched **8 of 128 channels** after a 40 s
download and a 40 MB cache. Region agreement alone is not enough.

- `iptv-org` — flat stems: `cnn.us`, `foxnews.us`
- `dotted-name` — internal dots: `NBC.Sports.Boston.HD.us2`
- `opaque` — makes no claim; always eligible

A lineup whose convention no listed source matches gets **no** automatic feed. That is
deliberate: nothing beats a slow download for ~6% coverage.

### Checking `health`

Use a **ranged GET**, never `HEAD`. epgshare01 returns 404 to `HEAD` for files that exist,
so a HEAD-based check marks every source dead:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' -r 0-200 "$URL"   # 200 or 206 = alive
```

## Deploying

Commit to `main`; GitHub Pages publishes from the repository root.

```bash
curl -sS https://epg.getstreamline.tv/manifest.json | jq .version
```
