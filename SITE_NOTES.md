# Site Notes

Internal notes on how this site is put together, for picking work back up later. Not a public README — just context for whoever (human or Claude) touches this repo next.

## What this is

Nathan Howland's personal site: static HTML/CSS/vanilla JS, no build step, no framework. Hosted on GitHub Pages from the `main` branch, served at the custom domain **nathanhowland.com** (via the `CNAME` file at the repo root — `nathanhowland.github.io` 301-redirects there).

## Deploying

There's no CI. Pushing to `main` is the deploy:

```
git add <files>
git commit -m "..."
git push origin main
```

GitHub Pages rebuilds within roughly a minute. **Always hard-refresh (Cmd+Shift+R) when checking the live site right after a push** — GitHub's CDN serves fresh content immediately, but browsers cache CSS/images aggressively by URL, and filenames here don't change between edits, so a normal reload can show stale assets and look like a broken deploy when it isn't. (This has caused confusion at least twice — the deploy was fine both times, the browser cache wasn't.)

## Page structure

- `index.html` — homepage: hero, About, Contact. No footer (removed site-wide).
- `music.html` — vertical list of project rows (album art left, description right, no boxes). Each row is a single `<a>` wrapping both the image and text, linking to a detail page in `music/`.
- `music/xoxo.html`, `music/laptop-ensemble.html` — individual project pages with the full original write-up + embeds (YouTube, SoundCloud).
- `blog.html` / `post.html` — blog index and post reader, driven by `blog/posts/posts.json` (see below).
- `projects.html` + `projects/*.html` — **hidden from nav site-wide but not deleted.** The nav `<li>` was removed from every page's `<ul class="nav-links">`; the files and their own internal nav still exist untouched. Re-link it by adding `<li class="nav-projects"><a href="projects.html">Projects</a></li>` back into each page's nav.

Nav order on every page: About · Contact · Music · Blog.

## Hero background

`.hero::before` in `css/style.css` renders `assets/coldvisions-watermark.png` as a low-opacity (currently `0.08`) full-bleed background image — no blend-mode tricks, because the current artwork is already a proper transparent PNG (white line art, real alpha channel). `assets/coldvisions.png` is kept as an identical copy for reference/history; only `coldvisions-watermark.png` is actually referenced by the CSS.

To swap the artwork: replace both files, then re-check opacity — a busy/high-contrast image needs a lower opacity to stay a background element instead of competing with the hero text (went from 0.16 → 0.08 last time the art changed to something busier).

`assets/background.png` is the *old* hero background image, no longer referenced anywhere. Left in place, unused.

## Blog system

`blog/posts/posts.json` is an array of post metadata, newest last (the page does `posts.slice().reverse()` to show newest first):

```json
{
  "title": "...",
  "date": "YYYY-MM-DD",
  "slug": "folder-name",
  "description": "one-line excerpt shown on the blog index",
  "cover": "filename.jpg"   // optional — shows a thumbnail on the blog index, path is blog/posts/<slug>/images/<cover>
}
```

Each post is its own folder: `blog/posts/<slug>/post.txt` + `blog/posts/<slug>/images/`. `post.html` fetches the slug from `?post=`, loads `posts.json` for metadata and `post.txt` for body content.

**`post.txt` format:** plain text, paragraphs separated by a blank line. A paragraph consisting of one of these tags is rendered specially instead of as text (defined in `post.html`'s inline script):

| Tag | Renders as |
|---|---|
| `[image: file.jpg]` or `[image: file.jpg \| caption]` | single image, optional caption |
| `[imagepair: a.jpg \| b.jpg]` | two images side by side |
| `[gallery: a.jpg, b.jpg, c.jpg, ...]` | responsive grid (3 cols desktop, 2 mobile) |
| `[video: path/to/file.mov]` or with `\| caption` | HTML5 `<video controls>`. Path is used as-is (not prefixed with the post's images dir) — use this for videos living elsewhere, e.g. `assets/video_media/...` |
| `[instagram: https://www.instagram.com/p/SHORTCODE/]` or with `\| caption` | Instagram's `/embed` iframe, extracted via regex on `/p/`, `/reel/`, or `/tv/` in the URL |

Image/gallery/imagepair paths are relative to `blog/posts/<slug>/images/`. Video and Instagram URLs are used verbatim.

### Adding a new post

1. Make the folder: `blog/posts/<slug>/images/`
2. Drop images in, **process them first** (see Image Processing below) — don't commit raw phone photos.
3. Write `post.txt` using the tags above.
4. Add an entry to `posts.json` (append at the end if it's the newest).
5. Test locally (`python3 -m http.server`, visit `post.html?post=<slug>`) before pushing.

## Image processing — read before touching any photos

Phone/camera photos landing in this repo are routinely 2–12MB and full sensor resolution. Always resize and compress before committing:

```python
from PIL import Image, ImageOps

MAX_DIM = 1600
im = Image.open(src_path)
im = ImageOps.exif_transpose(im)   # !! do this before anything else, see below
im = im.convert("RGB")
w, h = im.size
if max(w, h) > MAX_DIM:
    scale = MAX_DIM / max(w, h)
    im = im.resize((round(w*scale), round(h*scale)), Image.LANCZOS)
im.save(dst_path, "JPEG", quality=82, optimize=True)
```

**`ImageOps.exif_transpose()` is not optional.** iPhone/camera photos often store rotation as an EXIF orientation tag rather than physically rotating pixels — the file itself is sideways and relies on that tag to display upright. If you resize/re-save with plain PIL without calling `exif_transpose()` first, the new file has no orientation tag and **will display sideways everywhere**, even though it looked fine before. This actually happened once this session (`icey.jpg` and `sam-and-bella.jpg` in the Yard post shipped rotated 90° because the first compression pass skipped this step) and had to be fixed in a follow-up commit. Always transpose first, every time, no exceptions.

Flat graphics (flyers, screenshots with text) can stay PNG if they need real transparency; check with `im.mode == "RGBA"` and `im.getchannel("A").getextrema()` — if alpha is uniformly 255, there's no real transparency and it's safe (and much smaller) to flatten to JPEG.

## Music page — adding a new project

`music.html` uses `.music-list` > `.music-row` (a single `<a>` per project, wrapping both `.music-row-art` image and `.music-row-text`). To add one:

1. Add album art to `assets/music/`.
2. Add a new `.music-row` block to `music.html` (copy an existing one).
3. Create `music/<slug>.html` (copy `music/xoxo.html` or `music/laptop-ensemble.html` as a template) — it uses `.page-header` + `.project-header-inner` (cover art + title) and `.project-media` blocks for embeds.

## Working directory gotcha (Bash tool)

The sandbox's Bash tool persists `cd` across calls within a session. If you `cd` into a subfolder (e.g. to run `file *` on a folder of new images) and forget to `cd` back, subsequent commands — including Python scripts using relative paths — silently operate on/create files in the wrong place. Happened once this session (a stray `noclass/blog/posts/...` got created). Prefer `cd /full/path && command`, or just use absolute paths, over a bare persistent `cd`.

## Loose ends / known state

- `assets/background.png`, `assets/xxprofile.jpg`, `assets/media` (a stray 1-byte file) — unused, left in place, not cleaned up.
- `projects.html` and `projects/*.html` — functional but unlinked from nav (see above).
- No build step, no package.json, no dependencies. Just push HTML/CSS/JS/images directly.
