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
- Google Drive file and folder links.
- Seyarabata file links.
- Transfer.it transfer links.
- Yandex Disk public links.
- MediaFire file links.
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
./fastget 'https://transfer.it/t/TRANSFER_ID'
./fastget 'https://disk.yandex.com.tr/d/PUBLIC_ID'
./fastget 'https://www.mediafire.com/file/FILE_ID/FILENAME/file'
./fastget 'https://www.mediafire.com/folder/FOLDER_KEY/FOLDER_NAME'
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
- `-p, --password VALUE`: password for Gofile, SwissTransfer, or Transfer.it.
- `--interface IFACE`: bind `aria2c` sockets to one interface. On the target
  MacBook Pro, `en9` is the 10GbE Thunderbolt Ethernet interface.
- `--parallel N`: number of resolved HTTP files to download in parallel. This
  is most useful for Gofile/SwissTransfer folders or multiple direct URLs.
- `-h, --help`: print usage.

Password can also be supplied as:

```bash
FASTGET_PASSWORD='secret' fastget 'https://gofile.io/d/CONTENT_ID'
FASTGET_PASSWORD='secret' fastget 'https://transfer.it/t/TRANSFER_ID'
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
For provider links such as Gofile, SwissTransfer, Seyarabata, Transfer.it,
Yandex Disk, or MediaFire, the script
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

Supported public folder formats:

```text
https://drive.google.com/drive/folders/FOLDER_ID
https://drive.google.com/drive/u/0/folders/FOLDER_ID
https://drive.google.com/folderview?id=FOLDER_ID
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
- For file links, preflights the Google download endpoint before starting
  `aria2c`. If Google returns a current warning form, `fastget` extracts that
  form action and uses the confirmed download URL. If Google returns a quota,
  access, or other HTML error page, `fastget` stops before `aria2c` so it does
  not save an HTML error page as the requested file.
- If the link works in your signed-in browser but fails anonymously, set either
  `FASTGET_GDRIVE_COOKIE` to a raw Google `Cookie:` header value or
  `FASTGET_GDRIVE_COOKIE_FILE` to a Netscape/curl cookie jar exported from that
  browser session. `fastget` uses those cookies for the Google Drive preflight
  requests and passes them to `aria2c`.
- On macOS, you can also copy Chrome's full `Copy as cURL` command to the
  clipboard and run `fastget --gdrive-cookie-from-clipboard URL`. The cookie is
  parsed locally from the clipboard, is not printed, and is saved to
  `~/.config/fastget/gdrive-cookie` with owner-only file permissions. Future
  Google Drive downloads automatically reuse that cached cookie until Google
  expires it, so the copy step is normally only needed when the browser session
  changes or a cached cookie stops working.
- `fastget` does not read Chrome's cookie database or macOS Keychain directly.
  Google account cookies are account session credentials, so the supported
  convenience path is an explicit clipboard import followed by local caching.
- For folder links, fetches the public folder HTML from Google Drive, extracts
  the embedded `_DRIVE_ivd` listing, skips subfolder and native Google Workspace
  entries, and appends one download job for each visible binary file in that
  listing.
- Folder filenames come from the public folder listing, so multi-file Drive
  folders keep useful names without requiring a separate `HEAD` request per
  item.

Limitations:

- This is for public or otherwise accessible file links.
- Google Drive quota, virus scan interstitials, permission pages, or account-only
  files can still block downloads. `fastget` can detect and report those cases,
  but it cannot bypass Google quota or private-file restrictions.
- Browser-only success usually means your browser is sending Google account
  cookies. Use `fastget --gdrive-cookie-from-clipboard URL`,
  `FASTGET_GDRIVE_COOKIE_FILE`, or `FASTGET_GDRIVE_COOKIE` when you
  intentionally want `fastget` to use that same account session.
- Google Drive folder support reads the files exposed in the initial public
  folder page. It is not recursive, and very large folders that Google loads
  dynamically may need resolver updates if not all items are embedded in that
  page.
- Native Google Docs/Sheets/Slides entries are not exported through this
  resolver. Downloadable binary files are the intended target.

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

### Transfer.it

Recognized hosts:

- `transfer.it`
- `www.transfer.it`

Supported format:

```text
https://transfer.it/t/TRANSFER_ID
```

Resolver behavior:

1. Extracts `TRANSFER_ID` from `/t/TRANSFER_ID`. Transfer.it's own JavaScript
   names this value `xh`.
2. Calls the Transfer.it/MEGA-style metadata API:

```text
POST https://bt7.api.mega.co.nz/cs?id=0
body: [{"a":"xi","xh":"TRANSFER_ID"}]
```

3. Reads the transfer title token from `t` when present. The title is
   base64url-decoded and used as the aria2 output filename.
4. Checks `pw`. If the link is password-protected, `fastget` requires `-p` or
   `FASTGET_PASSWORD`, derives the same PBKDF2-SHA256 password token used by the
   Transfer.it web client, and validates it with:

```text
POST https://bt7.api.mega.co.nz/cs?id=0
body: [{"a":"xv","xh":"TRANSFER_ID","pw":"PASSWORD_TOKEN"}]
```

5. For single-file transfers, fetches the file list:

```text
POST https://bt7.api.mega.co.nz/cs?id=0&x=TRANSFER_ID
body: [{"a":"f","c":1,"r":1}]
```

6. Selects the first non-folder node (`t == 0`) as the downloadable file node.
7. For multi-file transfers, uses the archive node `z` from metadata when the
   API provides it. The output filename is forced to `.zip` if needed.
8. Requests a signed CDN download URL:

```text
POST https://bt7.api.mega.co.nz/cs?id=0&x=TRANSFER_ID
body: [{"a":"g","n":"NODE_HANDLE","pt":1,"g":1,"ssl":1}]
```

9. Appends the URL-encoded filename to the signed URL and passes it to `aria2c`.

Important detail: Transfer.it's static page is only a JavaScript bootloader. The
actual direct URL is not in the HTML. `fastget` therefore uses the same
underlying API flow as the web client. The signed CDN URL accepted a ranged GET
during testing, so aria2 segmented downloads can use the aggressive 10GbE HTTP
profile.

Limitations:

- Transfer.it signed CDN URLs are generated during resolver time. Start the
  download immediately after resolution instead of storing those URLs for later.
- Multi-file Transfer.it links need the API to expose an archive node `z`. If a
  multi-file link lacks that archive node, `fastget` stops rather than silently
  downloading only one item.

### Yandex Disk

Recognized hosts:

- `disk.yandex.*`, including `disk.yandex.com.tr`
- `yadi.sk`
- `www.yadi.sk`

Supported formats:

```text
https://disk.yandex.com.tr/d/PUBLIC_ID
https://disk.yandex.ru/d/PUBLIC_ID
https://yadi.sk/d/PUBLIC_ID
```

Resolver behavior:

1. Uses the entire public URL as the Yandex `public_key`. This matters because
   Yandex accepts localized domains such as `disk.yandex.com.tr`, while the API
   may return a normalized `yadi.sk` public URL.
2. Calls the public resource metadata API:

```text
GET https://cloud-api.yandex.net/v1/disk/public/resources?public_key=PUBLIC_URL&limit=1
```

3. Reads:

- `type`
- `name`
- `size` when provided by the API
- checksum fields such as `md5` and `sha256` when provided by the API

4. Uses `name` as the aria2 output filename. If the resource is a directory,
   appends `.zip` because the public download endpoint returns an archive-style
   download for folder resources.
5. Calls the public download API:

```text
GET https://cloud-api.yandex.net/v1/disk/public/resources/download?public_key=PUBLIC_URL
```

6. Reads the signed `href` returned by the API.
7. Passes that signed Yandex downloader URL to `aria2c`.

Important detail: opening the browser page can redirect to a Yandex captcha.
`fastget` avoids the page entirely and uses the public Disk API, which returned
a signed `downloader.disk.yandex.ru` URL for the tested Turkish-domain link.
That URL redirected to Yandex storage with `Accept-Ranges: bytes`, so aria2 can
use segmented downloads.

Limitations:

- Yandex signed downloader URLs are generated during resolver time. Do not store
  them for later use.
- Private Yandex Disk resources, removed public links, quota errors, or API-side
  abuse checks can still block resolution.
- Folder support depends on Yandex continuing to return a downloadable archive
  from the public download endpoint.

### MediaFire

Recognized hosts:

- `mediafire.com`
- `www.mediafire.com`
- `m.mediafire.com`

Supported formats:

```text
https://www.mediafire.com/file/FILE_ID/FILENAME/file
https://www.mediafire.com/download/FILE_ID/FILENAME
https://www.mediafire.com/folder/FOLDER_KEY/FOLDER_NAME
```

Resolver behavior:

1. For single-file links, extracts the file ID from `/file/FILE_ID/...` or
   `/download/FILE_ID/...`.
2. Fetches the MediaFire landing page with a browser-like user agent.
3. Finds the real download anchor in the file page:

```text
<a ... id="downloadButton" ... href="https://downloadNNNN.mediafire.com/...">
```

4. HTML-decodes the `href` and verifies it is an HTTP(S) URL.
5. Reads the filename from the page's `optFileName` JavaScript value when
   present.
6. Falls back to the final path component of the direct download URL if the page
   does not expose `optFileName`.
7. Passes the signed `downloadNNNN.mediafire.com` URL to `aria2c` with the
   resolved filename as `-o`.
8. For folder links, extracts the folder key from `/folder/FOLDER_KEY/...`.
9. Calls the MediaFire public folder API for files:

```text
GET https://www.mediafire.com/api/1.5/folder/get_content.php?folder_key=FOLDER_KEY&content_type=files&chunk=N&response_format=json
```

10. Calls the same API with `content_type=folders` to discover subfolders.
11. Recursively expands subfolders. Top-level files keep their own names;
    subfolder files are prefixed with the subfolder path using underscores so
    filenames do not collide in the download directory.
12. Resolves each returned `links.normal_download` file page through the same
    landing-page-to-CDN flow used for single-file links.

Important detail: MediaFire CDN URLs are generated by the landing page and can
expire or vary by request. `fastget` deliberately resolves the page at runtime
and sends the fresh URL to aria2. Already-direct `downloadNNNN.mediafire.com`
URLs are not treated as provider links; they remain ordinary direct HTTP(S)
downloads. Folder API entries still point at normal MediaFire file pages, so
each file is resolved immediately before aria2 receives the final URL.

The tested link's CDN URL returned:

```text
Accept-Ranges: bytes
Content-Disposition: attachment; filename="Packages.exe"
```

That means the existing aggressive segmented HTTP profile can be used.

Limitations:

- MediaFire can change the landing page markup. If `downloadButton` disappears
  or moves behind JavaScript-only API calls, update `mediafire_direct_from_html`
  and this README together.
- MediaFire can change the public folder API response. If that happens, update
  `mediafire_resolve_folder_files`, `mediafire_resolve_folder_children`, and
  this README together.
- Cloudflare, regional blocks, removed files, password-protected files, or
  account-only files can still prevent resolution.
- Very large MediaFire folders are expanded before downloading. Each file still
  needs its own fresh CDN URL, so huge folders can take time to resolve before
  aria2 starts.

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

- `FASTGET_PASSWORD`: fallback password for Gofile, SwissTransfer, or
  Transfer.it.
- `FASTGET_INTERFACE`: optional aria2 socket binding interface, for example
  `en9` for the 10GbE Thunderbolt Ethernet port.
- `FASTGET_PARALLEL`: parallel HTTP file downloads after provider resolution.
  Default is `4` aggressive and `1` sane.
- `FASTGET_GDRIVE_COOKIE`: raw Google Drive `Cookie:` header value to use for
  Drive files that require your signed-in browser session.
- `FASTGET_GDRIVE_COOKIE_FILE`: path to a Netscape/curl cookie jar to use for
  Google Drive files. This is the safer option when exporting browser cookies,
  because `fastget` can also reuse any temporary Drive cookies from warning
  pages.
- `FASTGET_GDRIVE_COOKIE_FROM_CLIPBOARD`: set to `1` to parse a Google Drive
  cookie from a copied Chrome/Firefox cURL command on the macOS clipboard. This
  is the environment equivalent of `--gdrive-cookie-from-clipboard`.
- `FASTGET_GDRIVE_COOKIE_CACHE`: path where a clipboard-imported Google Drive
  cookie is cached for future Drive downloads. Default is
  `~/.config/fastget/gdrive-cookie`.
- `FASTGET_GDRIVE_COOKIE_CACHE_DISABLE`: set to `1`, `true`, `yes`, or `on` to
  prevent reading or writing the Google Drive cookie cache.
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

Transfer.it:

- `FASTGET_TRANSFERIT_API`: overrides the Transfer.it API base. Default is
  `https://bt7.api.mega.co.nz`. This is only here in case Transfer.it moves the
  API host used by the web client.

Yandex Disk:

- `FASTGET_YANDEXDISK_API`: overrides the Yandex Disk public resources API base.
  Default is `https://cloud-api.yandex.net/v1/disk/public/resources`.

MediaFire:

- `FASTGET_MEDIAFIRE_API`: overrides the MediaFire API base used for folder
  listing. Default is `https://www.mediafire.com/api/1.5`.

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
- `jq`: used to parse Gofile, SwissTransfer, Transfer.it, Yandex Disk, and
  MediaFire folder JSON.
- `shasum`: used to SHA-256 hash Gofile passwords.
- `base64`: used to encode SwissTransfer passwords.

Optional:

- `python3`: used to URL-decode Google Drive filename hints, parse public Google
  Drive folder listings and Google Drive warning/error pages, URL/base64url
  encode Transfer.it filenames, and derive Transfer.it password tokens. If it is
  missing, unprotected Transfer.it links still work, but filenames may fall back
  to encoded text. Google Drive folder links, Drive warning-page handling, and
  password-protected Transfer.it links require Python 3.

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
  `is_seyarabata`, `is_transferit`, `is_yandexdisk`, `is_mediafire`: provider
  detectors.
- `is_torrent_like`: detects magnet links and `.torrent` paths/URLs.
- `looks_like_source`: helps decide whether a second positional argument is a
  password or another source.
- `append_download`: adds resolved jobs to the internal arrays.
- `resolve_pixeldrain`: converts Pixeldrain share URLs to API file URLs.
- `gdrive_folder_entries_from_html`: parses the embedded `_DRIVE_ivd` data from
  public Google Drive folder pages and emits visible binary file IDs and names.
- `append_gdrive_file`: preflights a Drive file download, follows Google's
  current warning form when available, detects quota/access HTML pages, and adds
  the final file job.
- `resolve_gdrive`: extracts Drive file IDs, expands public Drive folder
  listings, and builds direct download URLs.
- `resolve_gofile`: creates a guest account, fetches content metadata, walks
  files recursively, and stores the required cookie.
- `resolve_swisstransfer`: fetches transfer metadata, generates password tokens
  when needed, and builds per-file API download URLs.
- `resolve_seyarabata`: fetches the preview page, extracts the filename and
  `/d/FILE_ID` endpoint, and lets aria2 receive the fresh signed redirect.
- `resolve_transferit`: calls Transfer.it's metadata, file-list, password
  validation, and signed download URL APIs, then passes the signed CDN URL to
  aria2.
- `resolve_yandexdisk`: calls the Yandex Disk public metadata and download APIs,
  then passes the signed downloader URL to aria2.
- `resolve_mediafire`: fetches the MediaFire landing page, extracts the
  `downloadButton` direct CDN URL and filename, then passes the direct URL to
  aria2.
- `resolve_mediafire_folder`: expands a MediaFire folder link into file pages
  using the public folder API.
- `mediafire_resolve_folder_files`, `mediafire_resolve_folder_children`,
  `mediafire_resolve_folder_tree`: list folder contents, walk subfolders, and
  call the single-file MediaFire resolver for each file.
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

- Provider APIs can change. If Gofile, SwissTransfer, Transfer.it, or another
  provider changes response shape or token requirements, update the resolver and
  this README together.
- Google Drive can still block files that require login, exceed quota, show
  non-download interstitial pages, or hide folder contents from the public HTML
  listing. `fastget` reports known Drive quota/access pages; it does not bypass
  those server-side limits.
- Seyarabata support depends on the current `/t/FILE_ID` preview page and
  `/d/FILE_ID` redirect behavior.
- Transfer.it support depends on the current `xi`, `f`, `xv`, and `g` API
  actions exposed by the Transfer.it web client.
- Yandex Disk support depends on the public resources API and its signed
  download `href` response.
- MediaFire support depends on the current landing page exposing a
  `downloadButton` anchor with the signed CDN URL and on the current public
  folder API for folder links.
- Password support is implemented for Gofile, SwissTransfer, and Transfer.it.
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
- `resolve_transferit`
- `resolve_yandexdisk`
- `resolve_mediafire`

## License

See [LICENSE](./LICENSE).
