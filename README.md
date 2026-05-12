# fastget

`fastget` is a macOS-friendly Bash wrapper around `aria2c`. It keeps the
original fast segmented download behavior for ordinary HTTP(S) files and adds
resolvers for common share hosts that do not expose a direct download URL in the
browser link.

Current supported input types:

- Direct HTTP(S) URLs.
- Pixeldrain share links.
- Gofile folder/file links.
- SwissTransfer transfer links.
- Google Drive file links.
- Seyarabata file links.
- Magnet links.
- Local `.torrent` files.
- Remote HTTP(S) `.torrent` URLs.

The script is intended to be a single executable Bash file. It does not install
packages, create global config, or require a Python virtual environment.

## Quick Start

Install dependencies:

```bash
brew install aria2 jq
```

Run downloads from this repo:

```bash
./fastget 'https://example.com/file.zip'
./fastget -o file.zip 'https://example.com/download'
./fastget 'https://pixeldrain.com/u/FILE_ID'
./fastget 'https://drive.google.com/file/d/FILE_ID/view'
./fastget -p 'secret' 'https://gofile.io/d/CONTENT_ID'
./fastget -p 'secret' 'https://www.swisstransfer.com/d/TRANSFER_ID'
./fastget 'https://seyarabata.com/FILE_ID'
./fastget --interface en9 'https://example.com/huge-file.bin'
./fastget --parallel 6 'https://example.com/a.bin' 'https://example.com/b.bin'
./fastget 'magnet:?xt=urn:btih:...'
./fastget ./something.torrent
```

If `fastget` is on your `PATH`, use `fastget` instead of `./fastget`.

## Command Syntax

```bash
fastget [--sane] [-o output] [-p password] <url-or-torrent> [more...]
```

Options:

- `--sane`: use a less aggressive connection profile.
- `--aggressive`: explicitly use the default aggressive profile.
- `-o, --output NAME`: force the output filename when exactly one HTTP file is
  downloaded.
- `-p, --password VALUE`: password for Gofile or SwissTransfer.
- `--interface IFACE`: bind `aria2c` sockets to one interface. On the target
  MacBook Pro, `en9` is the 10GbE Thunderbolt Ethernet interface.
- `--parallel N`: number of resolved HTTP files to download in parallel. This
  is most useful for Gofile/SwissTransfer folders or multiple direct URLs.
- `-h, --help`: print usage.

Password can also be supplied as:

```bash
FASTGET_PASSWORD='secret' fastget 'https://gofile.io/d/CONTENT_ID'
```

For compatibility with the Telegram bot command style, this also works when the
first argument is a password-capable provider and the second argument does not
look like another URL/file source:

```bash
fastget 'https://www.swisstransfer.com/d/TRANSFER_ID' 'secret'
```

## What It Does

At a high level, `fastget` turns each user input into one or more concrete
download jobs, then runs `aria2c` for each job.

For direct URLs, magnets, and torrents, the input is already usable by `aria2c`.
For provider links such as Gofile, SwissTransfer, or Seyarabata, the script
first resolves the provider page/API, extracts direct file URLs or provider
download endpoints, chooses output filenames when available, attaches any
required cookies or tokens, then hands the final URLs to `aria2c`.

The script keeps HTTP downloads and BitTorrent downloads separate because they
need different `aria2c` options. HTTP downloads benefit from output names,
content-disposition handling, conditional requests, user-agent headers, and
segmented range requests. Torrent and magnet downloads should let torrent
metadata control file paths and need DHT, peer exchange, trackers, and no
seeding.

## How Inputs Are Processed

For every input argument:

1. The script checks the hostname or URI type.
2. If it recognizes a provider, it runs that provider resolver.
3. The resolver appends one or more entries to internal arrays:
   `DL_URLS`, `DL_NAMES`, `DL_COOKIES`, `DL_ORIGINS`, and `DL_PROVIDERS`.
4. HTTP entries are batched together and downloaded through one aria2 input file
   when `FASTGET_PARALLEL` is greater than 1.
5. Torrent/magnet entries flush the HTTP batch and then run through the torrent
   option path.
6. `download_one` chooses HTTP options or torrent options based on the final URL.
7. `aria2c` performs the actual download.

This means one share link may become many downloads. For example, a Gofile
folder containing five files resolves to five aria2 jobs, with up to
`FASTGET_PARALLEL` of those jobs active at once in the aggressive profile.

## Provider Behavior

### Direct HTTP(S)

Direct URLs are passed to `aria2c` with the HTTP option set.

Key behavior:

- Supports segmented downloads using `--split`, `--max-connection-per-server`,
  `--min-split-size`, and `--piece-length`.
- Sends `Accept-Encoding: identity` to avoid compressed transfer surprises.
- Sends a browser-like user agent by default.
- Respects `-o` only when one final HTTP download is being performed.

### Pixeldrain

Recognized host:

- `pixeldrain.com`

Supported share format:

```text
https://pixeldrain.com/u/FILE_ID
```

Resolver behavior:

- Extracts `FILE_ID` from `/u/FILE_ID`.
- Converts it to:

```text
https://pixeldrain.com/api/file/FILE_ID
```

- Passes that direct API URL to `aria2c`.

Already-direct Pixeldrain API links under `/api/file/...` are accepted as-is.

### Google Drive

Recognized hosts:

- `drive.google.com`
- `drive.usercontent.google.com`

Supported public file formats:

```text
https://drive.google.com/file/d/FILE_ID/view
https://drive.google.com/open?id=FILE_ID
https://drive.google.com/uc?id=FILE_ID
https://drive.usercontent.google.com/download?id=FILE_ID
```

Resolver behavior:

- Extracts the file ID from `/file/d/...` or from the `id=` query parameter.
- Builds a direct download URL:

```text
https://drive.usercontent.google.com/download?id=FILE_ID&export=download&confirm=t
```

- Attempts a `HEAD` request to learn the filename from
  `Content-Disposition`.
- If the filename is learned, it is passed to `aria2c` with `-o`.
- If the filename cannot be learned, `aria2c` chooses the name.

Limitations:

- This is for public or otherwise accessible file links.
- Google Drive quota, virus scan interstitials, permission pages, or account-only
  files can still block downloads.

### Gofile

Recognized host:

- `gofile.io`

Supported formats:

```text
https://gofile.io/d/CONTENT_ID
https://gofile.io/CONTENT_ID
```

Resolver behavior:

1. Creates a guest account with:

```text
POST https://api.gofile.io/accounts
```

2. Reads the returned account token.
3. Calls the content API:

```text
GET https://api.gofile.io/contents/CONTENT_ID
```

4. Sends:

```text
Authorization: Bearer <account_token>
X-Website-Token: 4fd6sg89d7s6
```

5. If a password was supplied, hashes it with SHA-256 and sends it as the
   provider expects.
6. Recursively walks returned `children`.
7. Appends every item whose type is `file` and has a `link`.
8. Downloads each file URL with this cookie header:

```text
Cookie: accountToken=<account_token>
```

Notes:

- The website token is kept in the script because the Telegram bot used the same
  value, based on common downloader behavior.
- Override it with `FASTGET_GOFILE_WEBSITE_TOKEN` if Gofile changes it.
- Password-protected Gofile links require `-p` or `FASTGET_PASSWORD`.

### SwissTransfer

Recognized host:

- `swisstransfer.com`

Supported format:

```text
https://www.swisstransfer.com/d/TRANSFER_ID
```

Resolver behavior:

1. Extracts the transfer/folder ID from the last path segment.
2. Calls:

```text
GET https://www.swisstransfer.com/api/links/TRANSFER_ID
```

3. Checks whether the link is expired or out of download credits.
4. Reads:

- `UUID`
- `downloadHost`
- `container.files[]`
- `needPassword`

5. If the transfer needs a password:

- Base64-encodes the password.
- Calls `generateDownloadToken` for each file.
- Appends the returned token as `?token=...`.

6. Builds each file URL:

```text
<downloadHost>/api/download/<TRANSFER_ID>/<FILE_UUID>
```

7. Downloads each resolved file URL with `aria2c`.

Password-protected SwissTransfer links require `-p` or `FASTGET_PASSWORD`.

### Seyarabata

Recognized hosts:

- `seyarabata.com`
- `www.seyarabata.com`

Supported formats:

```text
https://seyarabata.com/FILE_ID
https://seyarabata.com/t/FILE_ID
https://seyarabata.com/d/FILE_ID
```

Resolver behavior:

1. Extracts the file ID from the root path, `/t/FILE_ID`, or `/d/FILE_ID`.
2. Fetches the preview page:

```text
GET https://seyarabata.com/t/FILE_ID
```

3. Reads the visible filename from the preview title.
4. Reads the download button href when present.
5. Falls back to:

```text
https://seyarabata.com/d/FILE_ID
```

6. Sends that provider download endpoint to `aria2c`, with the preview filename
   as `-o` when it was found.

Important detail: `https://seyarabata.com/d/FILE_ID` returns a fresh signed
redirect to `dl.seyarabata.com`. The final signed URL accepted a range GET
during testing, but returned 404 to HEAD. Because of that, `fastget` deliberately
passes the `/d/FILE_ID` endpoint to aria2 instead of freezing the signed URL
found during resolution.

Already-signed `dl.seyarabata.com/...` URLs are treated as ordinary direct URLs
instead of Seyarabata provider links.

### Magnet Links

Recognized format:

```text
magnet:?xt=urn:btih:...
```

Behavior:

- Passed directly to `aria2c`.
- Uses the torrent option set.
- Enables DHT and peer exchange.
- Adds public trackers by default.
- Sets `--seed-time=0` so the process stops after download completion.
- Ignores `-o` because torrent metadata controls output paths.

### Torrent Files

Supported:

```bash
fastget ./file.torrent
fastget 'https://example.com/file.torrent'
```

Behavior:

- Local `.torrent` paths must exist.
- Remote `.torrent` URLs are passed to `aria2c`.
- Uses the torrent option set.
- Uses `--follow-torrent=mem` when supported, so remote torrent files are parsed
  without keeping the `.torrent` payload as the primary output.
- Ignores `-o` because torrent metadata controls output paths.

## Aria2 Profiles

The script has two profiles. The default aggressive profile is tuned for the
target machine this script is maintained on: Apple M4 Max, 128 GB RAM, 10GbE
over `en9`, default route MTU 9000.

Default aggressive profile:

```text
ARIA_CONN=<detected aria2 per-server cap, usually 16>
ARIA_SPLIT=64
ARIA_CHUNK=16M
ARIA_CACHE=2048M
ARIA_FILE_ALLOCATION=trunc
FASTGET_PARALLEL=4
```

Sane profile:

```text
ARIA_CONN=16
ARIA_SPLIT=16
ARIA_CHUNK=4M
ARIA_CACHE=512M
ARIA_FILE_ALLOCATION=none
FASTGET_PARALLEL=1
```

`aria2c` usually caps `--max-connection-per-server` at 16. `fastget` detects the
installed cap and clamps `ARIA_CONN` if needed.

The aggressive profile deliberately uses a much larger disk cache than aria2's
default. `aria2c` accepts `K` and `M` suffixes for `--disk-cache`, so the script
uses `2048M` instead of `2G`.

Shared defaults:

```text
ARIA_TRIES=0
ARIA_WAIT=5
ARIA_CONNECT_TIMEOUT=15
ARIA_TIMEOUT=60
ARIA_SUMMARY_INTERVAL=5
ARIA_NO_CONF=true
ARIA_DISABLE_IPV6=false
ARIA_BT_STOP_TIMEOUT=300
ARIA_BT_MAX_PEERS=100
```

`ARIA_TRIES=0` means unlimited retry attempts in aria2.

## 10GbE / Jumbo Frame Notes

The local machine checked during this tuning pass had:

```text
CPU: Apple M4 Max
Memory: 128 GB
Default route: en9
en9 media: 10Gbase-T full-duplex
en9 MTU: 9000
aria2: 1.37.0 with Async DNS, HTTPS, BitTorrent, Metalink, and mmap support
```

`fastget` does not set MTU. TCP uses the route/interface MTU selected by macOS.
For this machine the default IPv4 route was already using `en9` at MTU 9000, so
ordinary `fastget URL` traffic should use the jumbo-frame 10GbE path.

When Wi-Fi or VPN routing is also active, force the 10GbE interface explicitly:

```bash
./fastget --interface en9 'https://example.com/huge-file.bin'
FASTGET_INTERFACE=en9 ./fastget 'https://example.com/huge-file.bin'
```

The script also raises its soft file descriptor limit to
`FASTGET_ULIMIT_NOFILE=8192` by default. The checked shell had a soft limit of
256 and an unlimited hard limit, so raising the limit inside the process is
important for parallel provider folders and aggressive segmented downloads.

The macOS TCP autotune buffer values observed were 4 MiB max receive/send
autotune buffers. With aria2's 16 per-server connection cap, this is generally
enough to fill 10GbE on local/low-latency paths. Very high-latency WAN paths may
still be limited by remote server policy, TCP windows, packet loss, CDN region,
or provider throttling.

## Environment Variables

General:

- `FASTGET_PASSWORD`: fallback password for Gofile or SwissTransfer.
- `FASTGET_INTERFACE`: optional aria2 socket binding interface, for example
  `en9` for the 10GbE Thunderbolt Ethernet port.
- `FASTGET_PARALLEL`: parallel HTTP file downloads after provider resolution.
  Default is `4` aggressive and `1` sane.
- `FASTGET_ULIMIT_NOFILE`: runtime soft file descriptor target. Default is
  `8192`.
- `FASTGET_USER_AGENT`: HTTP user agent. Default is `Mozilla/5.0`.
- `FASTGET_CONNECT_TIMEOUT`: API connection timeout for `curl`. Default is `20`.
- `FASTGET_API_TIMEOUT`: total API timeout for `curl`. Default is `60`.

HTTP/aria2 tuning:

- `ARIA_CONN`: max connections per server before cap clamping.
- `ARIA_SPLIT`: split count.
- `ARIA_CHUNK`: min split size and piece length.
- `ARIA_CACHE`: aria2 disk cache size.
- `ARIA_FILE_ALLOCATION`: file allocation method. Default is `trunc` aggressive
  and `none` sane.
- `ARIA_TRIES`: max tries.
- `ARIA_WAIT`: seconds between retries.
- `ARIA_CONNECT_TIMEOUT`: aria2 connect timeout. Default is `15`.
- `ARIA_TIMEOUT`: aria2 socket timeout after connection. Default is `60`.
- `ARIA_SUMMARY_INTERVAL`: aria2 progress summary interval. Default is `5`.
- `ARIA_NO_CONF`: when `true`, passes `--no-conf=true` so hidden aria2 config
  cannot throttle fastget. Default is `true`.
- `ARIA_DISABLE_IPV6`: when `true`, passes `--disable-ipv6=true`. Default is
  `false`.

Gofile:

- `FASTGET_GOFILE_WEBSITE_TOKEN`: overrides the static Gofile website token.

BitTorrent:

- `ARIA_BT_TRACKERS`: comma-separated additional trackers.
- `ARIA_BT_STOP_TIMEOUT`: stop a torrent if download speed is zero for this many
  consecutive seconds.
- `ARIA_BT_MAX_PEERS`: max peers per torrent.

Example:

```bash
ARIA_CONN=8 ARIA_SPLIT=8 ./fastget --sane 'https://example.com/file.zip'
FASTGET_INTERFACE=en9 FASTGET_PARALLEL=6 ./fastget 'https://example.com/a.bin' 'https://example.com/b.bin'
ARIA_BT_STOP_TIMEOUT=600 ./fastget 'magnet:?xt=urn:btih:...'
```

## Dependencies

Required for all downloads:

- `bash`
- `aria2c`

Required for provider resolvers:

- `curl`: used for API calls and Google Drive filename detection.
- `jq`: used to parse Gofile and SwissTransfer JSON.
- `shasum`: used to SHA-256 hash Gofile passwords.
- `base64`: used to encode SwissTransfer passwords.

Optional:

- `python3`: used only to URL-decode Google Drive filename hints. If missing,
  the undecoded filename is used.

On macOS, `curl`, `shasum`, and `base64` are normally already present. Install
the common missing tools with:

```bash
brew install aria2 jq
```

## Important Implementation Notes

The main file is [fastget](./fastget).

Key functions:

- `usage`: command help and documented interface.
- `need_tool`: dependency checks with install hints.
- `raise_fd_limit`: raises the process soft file descriptor limit for aggressive
  parallel downloads when the shell starts with a small limit.
- `url_host`, `url_path`, `query_param`: small URL helpers implemented in Bash.
- `host_matches`: safe domain matching for provider detection.
- `is_pixeldrain`, `is_gofile`, `is_swisstransfer`, `is_gdrive`,
  `is_seyarabata`: provider detectors.
- `is_torrent_like`: detects magnet links and `.torrent` paths/URLs.
- `looks_like_source`: helps decide whether a second positional argument is a
  password or another source.
- `append_download`: adds resolved jobs to the internal arrays.
- `resolve_pixeldrain`: converts Pixeldrain share URLs to API file URLs.
- `resolve_gdrive`: extracts Drive file IDs and builds direct download URLs.
- `resolve_gofile`: creates a guest account, fetches content metadata, walks
  files recursively, and stores the required cookie.
- `resolve_swisstransfer`: fetches transfer metadata, generates password tokens
  when needed, and builds per-file API download URLs.
- `resolve_seyarabata`: fetches the preview page, extracts the filename and
  `/d/FILE_ID` endpoint, and lets aria2 receive the fresh signed redirect.
- `resolve_input`: dispatches one user argument to the right resolver.
- `detect_conn_cap`: reads the local `aria2c` help output and detects the max
  connection cap.
- `has_opt`: checks whether the installed `aria2c` supports an option before
  using it.
- `download_one`: chooses HTTP or torrent options and runs `aria2c`.
- `batch_add_http`, `flush_http_batch`, `download_http_batch`: collect resolved
  HTTP downloads and feed them to aria2 input-file mode for parallel
  multi-file downloads.

Internal arrays:

- `DL_URLS`: final URLs or torrent/magnet inputs passed to `aria2c`.
- `DL_NAMES`: optional output names for HTTP downloads.
- `DL_COOKIES`: optional cookie strings, currently used by Gofile.
- `DL_ORIGINS`: original user inputs. Kept for future diagnostics.
- `DL_PROVIDERS`: provider labels used in status output.
- `BATCH_URLS`, `BATCH_NAMES`, `BATCH_COOKIES`, `BATCH_PROVIDERS`: temporary
  HTTP batch arrays used before the next torrent/magnet boundary or final
  flush.

Do not collapse these arrays into a single string format unless you also handle
tabs, spaces, quotes, and newlines in filenames and URLs. Bash arrays are used
intentionally.

## Update Guide for Future AI/Developers

When adding a provider:

1. Add a detector function, usually using `host_matches`.
2. Add a resolver function named `resolve_<provider>`.
3. Make the resolver call `append_download` once per final downloadable file.
4. Include a filename if the provider returns one.
5. Include a cookie string if the final download needs one.
6. Add the provider to `resolve_input`.
7. Add README documentation for URL formats, API calls, auth headers, and
   limitations.
8. Run the verification commands below.

Resolver rules:

- Keep provider API calls in the resolver, not in `download_one`.
- Keep final downloads in `download_one`, not in the resolver.
- Prefer structured JSON parsing with `jq`.
- Avoid parsing JSON with `sed`, `awk`, or `grep`.
- Keep passwords out of status output.
- If a provider can resolve multiple files, append multiple jobs rather than
  trying to combine them inside the resolver. The HTTP batch layer handles
  parallel aria2 execution.
- If a provider requires a changing token, add an environment override for it.

Torrent rules:

- Do not pass `-o` for magnets or torrents.
- Do not add HTTP-only options to torrent downloads unless `aria2c` explicitly
  supports that behavior for torrents.
- Keep `--seed-time=0` unless the project intentionally changes from a download
  tool into a seeding tool.

Bash compatibility rules:

- The script is tested with Homebrew Bash 5 and macOS `/bin/bash` 3.2.
- Avoid Bash features newer than 3.2 unless you change the shebang and document
  the new requirement.
- Use arrays for arguments passed to `aria2c`.
- Quote variable expansions.
- Avoid building command strings and `eval`.
- Keep the aria2 input-file writer inside `download_http_batch` careful and
  boring: one URL line, then indented per-download options such as `out=` and
  `header=`.

## Verification

Run these after editing:

```bash
bash -n fastget
/bin/bash -n fastget
./fastget --help
git diff --check
```

Small local HTTP smoke test:

```bash
php -S 127.0.0.1:8765
```

In another terminal:

```bash
cd /tmp
/path/to/fastget --sane -o fastget-test-readme.txt http://127.0.0.1:8765/README.md
rm -f /tmp/fastget-test-readme.txt /tmp/fastget-test-readme.txt.aria2
```

Parallel local HTTP smoke test:

```bash
cd /tmp
/path/to/fastget --parallel 2 http://127.0.0.1:8765/README.md http://127.0.0.1:8765/LICENSE
rm -f /tmp/README.md /tmp/README.md.aria2 /tmp/LICENSE /tmp/LICENSE.aria2
```

Provider tests require real links. Prefer small test files because `fastget`
will download the full resolved file.

For a large provider link where you need to test only the resolver path, export
a fake `aria2c` function in a throwaway shell so the script prints the final
arguments without downloading. Keep this as a manual diagnostic rather than
part of normal use.

## Known Limitations

- Provider APIs can change. If Gofile or SwissTransfer changes response shape or
  token requirements, update the resolver and this README together.
- Google Drive can still block files that require login, exceed quota, or show
  non-download interstitial pages.
- Seyarabata support depends on the current `/t/FILE_ID` preview page and
  `/d/FILE_ID` redirect behavior.
- Password support is implemented only for Gofile and SwissTransfer.
- The script downloads files only. It does not upload to Telegram, split files,
  or mirror media. That behavior belongs to the separate Telegram bot project.
- No automatic cleanup is performed beyond what `aria2c` normally does.
- A single remote server can still be the bottleneck. The script asks aria2 for
  the local maximum, but remote CDNs and file hosts may cap per-IP, per-file, or
  per-account throughput.

## Relationship to the Telegram Bot

The provider logic was ported from:

```text
/Users/rhcp/Documents/PROJECTS/Filebot/mirrorbot.py
```

The bot does extra Telegram-specific work such as status messages, upload,
splitting large files with 7z, and cleanup of job directories. `fastget` only
performs local downloads.

Provider parity brought into `fastget`:

- `resolve_pixeldrain`
- `resolve_gofile`
- `resolve_swisstransfer`
- `resolve_gdrive`
- magnet and `.torrent` handling through `aria2c`

Provider support added directly to `fastget` after the bot port:

- `resolve_seyarabata`

## License

See [LICENSE](./LICENSE).
