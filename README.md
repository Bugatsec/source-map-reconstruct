# Source Maps Reconstructer

![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white)
![Mozilla Source Map](https://img.shields.io/badge/Mozilla-source--map-FF7139?logo=mozilla&logoColor=white)
![License](https://img.shields.io/badge/license-ISC-blue)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20WSL-lightgrey)
![Security Research](https://img.shields.io/badge/use-security%20research-red)

**`source-map-reconstruct`** is a tiny wrapper script around Mozilla's `source-map` npm package. It is made for security researchers, bug bounty hunters, and JavaScript reviewers who want to analyze exposed source maps without repeating the same manual steps every time.

You give it the JavaScript URL and the source map URL. It downloads both files, stores them under a folder scoped to the target host and bundle path, reconstructs a cleaner readable file from `sourcesContent`, writes each original source to its own file under `raw/`, and saves the original URLs in `urls.txt` so you can remember exactly where each file came from later.


## Features

- Downloads a JavaScript bundle from `-js <url>`, or uses a local file path.
- Downloads its source map from `-map <url>`, uses a local file path, or **auto-discovers** it from a `//# sourceMappingURL=` comment or `SourceMap` response header in the JS: including inline `data:` maps that need no second request.
- Creates a folder scoped per bundle under the target host, for example `example.com/_next/static/chunks/main_a1b2c3d4/`.
- Saves the original `.js` and `.map` files.
- Reconstructs readable source content into `reconstructed-<name>.js` (all sources concatenated).
- Writes each source as its own file under `raw/`, preserving the original `webpack://` folder structure (`raw/src/auth.js`, `raw/src/api/client.js`, etc.).
- Writes `raw/MANIFEST.txt` mapping every original source path to the file it was written to.
- Appends every run to `urls.txt` with the reconstructed filename, JS URL, and map URL.
- Optionally appends structured JSON-lines records to a `--jsonl` file for pipeline/dashboard integration.
- Supports custom request headers (`--header`), proxy routing (`--proxy`), custom User-Agent (`--user-agent`), and configurable timeout (`--timeout`).
- Handles gzip, deflate, and brotli response encoding automatically.
- Retries transient network errors up to 3 times with exponential backoff.
- Keeps multiple reconstructions organized and isolated — two bundles in the same directory on the same host never overwrite each other.

## Requirements

- Node.js 18 or newer
- npm 9 or newer
- zsh if you want the `map` alias

Install the project dependencies from `package.json`:

```sh
npm install
```

This installs Mozilla's `source-map` npm package. If you are setting the wrapper up manually from a fresh folder, install it directly with:

```sh
npm install source-map
```

## Usage

Run from the tool directory:

```sh
./map -js "https://example.com/static/app.js" -map "https://example.com/static/app.js.map"
```

With the zsh alias installed:

```sh
map -js "https://example.com/static/app.js" -map "https://example.com/static/app.js.map"
```

Let the tool discover the map automatically from the JS:

```sh
map -js "https://example.com/static/app.js"
```

Use a local map file you already have on disk:

```sh
map -js "https://example.com/static/app.js" -map ./app.js.map
```

Route through Burp Suite and attach an auth cookie:

```sh
map -js "https://example.com/static/app.js" \
    --proxy http://127.0.0.1:8080 \
    --insecure \
    --header "Cookie: session=abc123"
```

Write output to a specific directory and append to a shared jsonl log:

```sh
map -js "https://example.com/static/app.js" \
    -o ~/targets/example \
    --jsonl ~/targets/example/sourcemaps.jsonl
```

## Options

| Flag | Description |
|---|---|
| `-js <url\|path>` | URL or local path of the JS bundle. Omit to skip the download when you only need the map. |
| `-map <url\|path>` | URL or local path of the `.js.map` file. Omit to auto-discover from the JS. |
| `-o, --output <dir>` | Base output directory. Default: `./source-map` |
| `--header "N: v"` | Extra request header, repeatable. Use for cookies, tokens, or any auth the target requires. |
| `--proxy <url>` | HTTP proxy URL, e.g. `http://127.0.0.1:8080`. HTTPS targets use a CONNECT tunnel. |
| `--user-agent <ua>` | User-Agent string. Default: Chrome 124. |
| `--timeout <ms>` | Per-request timeout in milliseconds. Default: `30000`. |
| `--insecure` | Disable TLS certificate verification. Use for self-signed or lab targets. |
| `--no-raw` | Skip writing the per-source-file `raw/` tree. |
| `--no-combined` | Skip writing the combined `reconstructed-*.js` file. |
| `--jsonl <file>` | Append structured JSON-lines records for this run to a file. |
| `-h, --help` | Show usage. |

## Output

For this command:

```sh
map -js "https://example.com/static/app.js" -map "https://example.com/static/app.js.map"
```

The wrapper creates:

```text
source-map/
└── example.com/
    └── static/
        └── app_3f9a1b2c/
            ├── app.js
            ├── app.js.map
            ├── reconstructed-app.js
            ├── urls.txt
            └── raw/
                ├── MANIFEST.txt
                ├── src/
                │   ├── auth.js
                │   └── utils.js
                └── api/
                    └── client.js
```

The trailing `_3f9a1b2c` is a short hash of the full JS URL. It ensures that two bundles in the same directory: for example `_next/static/chunks/main.a1b2c3.js` and `_next/static/chunks/vendor.d4e5f6.js` — always get their own isolated folder and never overwrite each other's `raw/` files or `MANIFEST.txt`.

The `urls.txt` file looks like this:

```text
/* ==== filename: reconstructed-app.js ==== */
js:  https://example.com/static/app.js
map: https://example.com/static/app.js.map


/* ==== filename: reconstructed-vendor.js ==== */
js:  https://example.com/static/vendor.js
map: https://example.com/static/vendor.js.map
```

`raw/MANIFEST.txt` maps every entry in the sourcemap's `sources[]` to the file that was written:

```text
webpack://app/./src/auth.js       ->  src/auth.js
webpack://app/./src/utils.js      ->  src/utils.js
webpack://app/./api/client.js     ->  api/client.js
```

## Alias Setup

Add this to `~/.zshrc`:

```sh
alias map="$HOME/tool/source-map/map"
```

Then reload zsh:

```sh
source ~/.zshrc
```

For this copied repo path, update the alias to wherever you keep the repo.

## Notes

Source maps can only reconstruct original source content when the `.map` file includes `sourcesContent`. If `sourcesContent` is missing, the wrapper still saves the `.js`, `.map`, and `urls.txt`, and writes a reconstructed file listing the sources referenced by the map.

All source paths from the map are sanitized before anything is written to disk. Path traversal entries like `webpack://../../../../etc/passwd` are stripped of `..` segments and confirmed to stay inside the `raw/` output directory before writing. Sourcemap content comes from the target and is treated as untrusted input.

Use this only on assets you are authorized to inspect, such as your own applications, bug bounty targets, or security testing scopes where this kind of analysis is allowed.
