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
For provider links such as Gofile or SwissTransfer, the script first calls that
provider's API, extracts direct file URLs, chooses output filenames when
available, attaches any required cookies or tokens, then hands the final URLs to
`aria2c`.

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
4. After all inputs are resolved, each entry is downloaded with `download_one`.
5. `download_one` chooses HTTP options or torrent options based on the final URL.
6. `aria2c` performs the actual download.

This means one share link may become many downloads. For example, a Gofile
folder containing five files resolves to five `aria2c` runs.

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

The script has two profiles.

Default aggressive profile:

```text
ARIA_CONN=32
ARIA_SPLIT=32
ARIA_CHUNK=4M
ARIA_CACHE=512M
```

Sane profile:

```text
ARIA_CONN=16
ARIA_SPLIT=16
ARIA_CHUNK=2M
ARIA_CACHE=256M
```

`aria2c` usually caps `--max-connection-per-server` at 16. `fastget` detects the
installed cap and clamps `ARIA_CONN` if needed.

Shared defaults:

```text
ARIA_TRIES=0
ARIA_WAIT=5
ARIA_BT_STOP_TIMEOUT=300
ARIA_BT_MAX_PEERS=100
```

`ARIA_TRIES=0` means unlimited retry attempts in aria2.

## Environment Variables

General:

- `FASTGET_PASSWORD`: fallback password for Gofile or SwissTransfer.
- `FASTGET_USER_AGENT`: HTTP user agent. Default is `Mozilla/5.0`.
- `FASTGET_CONNECT_TIMEOUT`: API connection timeout for `curl`. Default is `20`.
- `FASTGET_API_TIMEOUT`: total API timeout for `curl`. Default is `60`.

HTTP/aria2 tuning:

- `ARIA_CONN`: max connections per server before cap clamping.
- `ARIA_SPLIT`: split count.
- `ARIA_CHUNK`: min split size and piece length.
- `ARIA_CACHE`: aria2 disk cache size.
- `ARIA_TRIES`: max tries.
- `ARIA_WAIT`: seconds between retries.

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
- `url_host`, `url_path`, `query_param`: small URL helpers implemented in Bash.
- `host_matches`: safe domain matching for provider detection.
- `is_pixeldrain`, `is_gofile`, `is_swisstransfer`, `is_gdrive`: provider
  detectors.
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
- `resolve_input`: dispatches one user argument to the right resolver.
- `detect_conn_cap`: reads the local `aria2c` help output and detects the max
  connection cap.
- `has_opt`: checks whether the installed `aria2c` supports an option before
  using it.
- `download_one`: chooses HTTP or torrent options and runs `aria2c`.

Internal arrays:

- `DL_URLS`: final URLs or torrent/magnet inputs passed to `aria2c`.
- `DL_NAMES`: optional output names for HTTP downloads.
- `DL_COOKIES`: optional cookie strings, currently used by Gofile.
- `DL_ORIGINS`: original user inputs. Kept for future diagnostics.
- `DL_PROVIDERS`: provider labels used in status output.

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
  trying to combine them into one `aria2c` call.
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

Provider tests require real links. Prefer small test files because `fastget`
will download the full resolved file.

## Known Limitations

- Provider APIs can change. If Gofile or SwissTransfer changes response shape or
  token requirements, update the resolver and this README together.
- Google Drive can still block files that require login, exceed quota, or show
  non-download interstitial pages.
- Password support is implemented only for Gofile and SwissTransfer.
- The script downloads files only. It does not upload to Telegram, split files,
  or mirror media. That behavior belongs to the separate Telegram bot project.
- No automatic cleanup is performed beyond what `aria2c` normally does.

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

## License

See [LICENSE](./LICENSE).
