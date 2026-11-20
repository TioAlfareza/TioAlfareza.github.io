# H5P Listening Materials - static site

Two static pages that play the exported `.h5p` packages without an H5P server: a landing page
that introduces the product, and a player page for one unit.

## Run locally

The page must be served over http. Opening `index.html` directly with `file://` will not work,
because the player fetches `content/*/h5p.json` with `fetch()`.

```
cd h5p-preview
python -m http.server 8099
```

Then open http://127.0.0.1:8099/

Any static server works (`npx serve .`, `php -S`, nginx, etc.). No build step, no database, no PHP.

## How it is laid out

```
index.html                  landing page: hero, unit cards, about
unit.html                   player page, one unit per load (?u=<slug>)
assets/h5p-player/          h5p-standalone player (copied from npm dist)
content/unit-1-narrative/   extracted .h5p package
content/unit-2-argumentative/
content/unit-3-hortatory/
content/unit-4-discussion/
```

Each unit is the raw `.h5p` archive extracted in place. The site reads `h5p.json` and
`content/content.json` from that folder, so replacing a unit means replacing that folder.
Card thumbnails are the book cover images already inside each package
(`content/<slug>/content/images/`, path taken from `bookCover.coverMedium` in `content.json`).

## Adding or replacing a unit

1. Extract the `.h5p` file (it is a zip) into `content/<slug>/`.
2. Add an entry to the `UNITS` array in **both** `index.html` and `unit.html`:

```js
{ slug:'unit-5-analytical', topic:'Analytical Exposition', title:'Your unit title' },
```

3. Copy a unit card in `index.html` and point its `href` at `unit.html?u=unit-5-analytical`.

## Notes on the player

- `h5p-standalone` 3.8.2 from npm. Its `dist/` folder is copied into `assets/h5p-player/`.
- The constructor is read once at page load (`const Player = H5PStandalone && H5PStandalone.H5P`)
  because H5P's own init clears that global. For the same reason each unit loads in a fresh
  document (`unit.html?u=<slug>`) instead of remounting in place.
- **Paths must be absolute.** The player builds asset URLs as `host + path + '/' + file`, with no
  separator between host and path. A relative `h5pJsonPath` like `content/unit-1` therefore
  produces `http://hostcontent/unit-1/...` and every image 404s. `unit.html` prepends `BASE`
  (the directory the page is served from) for this reason.
- Images live in `content/<slug>/content/images/`. If they are missing after an export, the
  `.h5p` was exported without them — re-export with "Include all content".
- **One library is patched.** `content/<slug>/H5P.Video-1.6/scripts/youtube.js` line 169 now reads
  `if (player.g) { player.g.style = ... }`. Upstream assumes `player.g` always exists and throws a
  `TypeError` in the console otherwise. Re-extracting a `.h5p` overwrites this patch, so re-apply it
  after replacing a unit.
- The videos are YouTube embeds (`youtube.com/embed/...`), so the page needs internet access.
  Offline they show an empty black frame, not an error.
- YouTube embeds fail with `Error 153` when the browser suppresses the referrer, and they fail
  with no player UI in headless or offscreen Chromium. Both were observed in the OpenCode preview
  browser, not in a normal desktop browser. Verified with `youtube.com/oembed` that the videos
  themselves are embeddable. If a video still fails in a normal browser, export it to `.mp4` and
  swap the video source in `content/<slug>/content/content.json`.
- **Playback depends on the YouTube video staying public.** A private or deleted video shows
  "This video is private" inside the player. Check the eight source videos with
  `https://www.youtube.com/oembed?url=https://www.youtube.com/watch?v=<id>&format=json` — a 403
  means it is not public.
- `node_modules/` is only needed to re-copy `dist/`. The site runs without it.

## Deploying

Upload `index.html`, `assets/` and `content/` to any static host. Total size is about 64 MB,
which matters for GitHub Pages (1 GB limit, fine) and for free Netlify/Vercel (100 GB bandwidth).
Serve over https so the published link is usable from a journal article.

## Missing

Unit 5 (Analytical Exposition) has no `.h5p` export, so it is not on the site. The folder
`Desktop\Workout\5 Analytical Exposition Text` contains only a .docx and page screenshots.

## Download links

`downloads/*.h5p` are copies of the original exports, one per unit, served as static files.
The landing page links each card to its file with a `download` attribute. Replace a file there
when you replace a unit's content.
