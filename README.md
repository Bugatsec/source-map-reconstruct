# Source Maps

![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933?logo=node.js&logoColor=white)
![Mozilla Source Map](https://img.shields.io/badge/Mozilla-source--map-FF7139?logo=mozilla&logoColor=white)
![License](https://img.shields.io/badge/license-ISC-blue)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20WSL-lightgrey)
![Security Research](https://img.shields.io/badge/use-security%20research-red)

Source Maps is a tiny wrapper script around Mozilla's `source-map` npm package. It is made for security researchers, bug bounty hunters, and JavaScript reviewers who want to analyze exposed source maps without repeating the same manual steps every time.

You give it the JavaScript URL and the source map URL. It downloads both files, stores them under a folder named after the target host, reconstructs a cleaner readable file from `sourcesContent`, and saves the original URLs in `urls.txt` so you can remember exactly where each file came from later.

Recommended repo name: `source-maps`. Other good options: `map-recall`, `sourcemap-reconstructor`, or `source-map-wrapper`.

## Features

- Downloads a JavaScript bundle from `-js <url>`.
- Downloads its source map from `-map <url>`.
- Creates a folder named after the URL host, for example `example.com/`.
- Saves the original `.js` and `.map` files.
- Reconstructs readable source content into `reconstructed-<name>.js`.
- Appends every run to `urls.txt` with the reconstructed filename, JS URL, and map URL.
- Keeps multiple reconstructions organized by host.
- Works well in Kali WSL with a simple `map` alias.

## Requirements

- Node.js 18 or newer
- npm 9 or newer
- zsh if you want the `map` alias

Install dependencies:

```sh
npm install
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

## Output

For this command:

```sh
map -js "https://example.com/static/app.js" -map "https://example.com/static/app.js.map"
```

The wrapper creates:

```text
source-maps/
├── map
├── package.json
├── package-lock.json
├── requirements.txt
├── README.md
└── example.com/
    ├── app.js
    ├── app.js.map
    ├── reconstructed-app.js
    └── urls.txt
```

The `urls.txt` file looks like this:

```text
/* ==== filename: reconstructed-app.js ==== */
js: https://example.com/static/app.js
map: https://example.com/static/app.js.map

/* ==== filename: reconstructed-vendor.js ==== */
js: https://example.com/static/vendor.js
map: https://example.com/static/vendor.js.map
```

## Alias Setup

Add this to `~/.zshrc` if the tool lives in `~/mytool/source-map`:

```sh
alias map="$HOME/mytool/source-map/map"
```

Then reload zsh:

```sh
source ~/.zshrc
```

For this copied repo path, update the alias to wherever you keep the repo.

## Notes

Source maps can only reconstruct original source content when the `.map` file includes `sourcesContent`. If `sourcesContent` is missing, the wrapper still saves the `.js`, `.map`, and `urls.txt`, and writes a reconstructed file listing the sources referenced by the map.

Use this only on assets you are authorized to inspect, such as your own applications, bug bounty targets, or security testing scopes where this kind of analysis is allowed.
