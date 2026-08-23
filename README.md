# Portfolio site

Jekyll source for a personal portfolio + weekly research-log site, built for
GitHub Pages (user site: `https://TanoojMadhuvan.github.io`).

## Structure

```
_config.yml         Site settings — title, social handles, collections, etc.
index.md            Home page (/)
about.md            /about
foundation.md       /foundation — static reference on the base RISC-V core
tracks.md           /tracks — index of all research tracks
blog.md             /blog — chronological feed of every post
resume.md           /resume

_tracks/             One file per research track (collection). Each file's
                      front matter (title, slug, summary, status, color) drives
                      the /tracks index, the track page header, and every post's
                      track badge — `color` is a hex string used to tint that
                      track's icon, badge, and card border.
_posts/               Weekly blog posts. Filename: YYYY-MM-DD-slug.md

_layouts/, _includes/, assets/css/main.scss   Theme (custom, no external framework)
_includes/track-icon.html    Inline SVG icon per track, keyed by slug
_includes/social-icon.html   Inline SVG icons for GitHub/LinkedIn/email
_includes/pipeline-diagram.html   The IF/ID/EX/MEM/WB SVG on the homepage
assets/images/        Post images (including optional post cover images)
assets/files/         Drop resume.pdf here for the /resume download button
```

**Theme:** Inter (body text) + JetBrains Mono (code, dates, badges, stat
numbers) loaded from Google Fonts. Two accent colors — orange (`--accent`,
CTAs/brand) and teal (`--accent2`, links/circuit accents) — plus a per-track
`color` used for that track's icon/badge/card. All of this lives in
`assets/css/main.scss` as CSS custom properties (`:root` for light,
`html[data-theme="dark"]` for dark) — edit values there to retheme.

## Adding a new weekly post

1. Create a file in `_posts/` named `YYYY-MM-DD-short-title.md` (the date
   controls sort order and must not be in the future, or Jekyll won't publish
   it by default).
2. Add front matter:

   ```yaml
   ---
   title: "Week N: Whatever you worked on"
   date: 2026-08-25
   track: branch-prediction   # must match a slug in _tracks/
   summary: "One sentence for the post-list preview."
   ---
   ```

   `track` must exactly match the `slug` of one of the files in `_tracks/`
   (`branch-prediction`, `out-of-order`, `ppa-analysis`, `verification`,
   `kv-cache`, `systolic-array`). The post will then automatically show up on
   that track's page, on `/blog`, and (if recent) on the homepage.

3. Write the body in Markdown. Code blocks get syntax highlighting via Rouge —
   use ` ```systemverilog ` or ` ```cpp ` as the fence language.
4. Embed inline images with standard Markdown, referencing files under
   `assets/images/`:

   ```markdown
   ![Alt text](/assets/images/branch-prediction/week1-waveform.png)
   ```

5. Optional: a full-width cover image at the top of the post (e.g. a waveform
   screenshot or RTL diagram), via extra front matter fields:

   ```yaml
   image: /assets/images/branch-prediction/week1-cover.png
   image_alt: "Waveform showing the predictor saturating after 3 taken branches"
   image_caption: "Optional caption shown under the image"   # optional
   ```

## Adding a new track

Only do this if you're starting an entirely new line of work (rare — the six
existing tracks are already scaffolded in `_tracks/`). Add a file
`_tracks/<slug>.md`:

```yaml
---
title: Track Name
slug: track-slug
summary: One sentence for the /tracks grid.
status: Active
color: "#4C8FCB"   # tints this track's icon, badge, and card border
---
Intro paragraph(s): what this track is and why you're doing it.
```

Then add `<slug>` to `tracks_order` in `_config.yml` so it appears on
`/tracks` in the right position, and add a matching `{% when "track-slug" %}`
case (with an inline SVG) to `_includes/track-icon.html` — otherwise it falls
back to a plain circle icon.

## Running locally

**This machine's setup:** a portable Ruby 3.2.11 + MSYS2/mingw-w64 toolchain
was installed (no admin rights needed) to `C:\Users\tanoo\ruby32\`, since
Ruby wasn't previously installed. To run the site:

```bash
export PATH="/c/Users/tanoo/ruby32/rubyinstaller-3.2.11-1-x64/bin:/c/Users/tanoo/ruby32/rubyinstaller-3.2.11-1-x64/msys64/ucrt64/bin:/c/Users/tanoo/ruby32/rubyinstaller-3.2.11-1-x64/msys64/usr/bin:$PATH"
bundle exec jekyll serve --livereload
```

Then open http://localhost:4000 — edits to content/layouts auto-reload in the
browser (via polling; the `wdm` native file-watcher gem isn't installed, see
note in `Gemfile`).

**Alternative — Docker** (no local Ruby install at all):

```bash
docker run --rm -it \
  -v "$PWD:/srv/jekyll" -v "$PWD/vendor/bundle:/usr/local/bundle" \
  -p 4000:4000 \
  jekyll/jekyll:latest \
  jekyll serve --livereload
```

**On a different machine** (fresh Ruby install): run `bundle install` once,
then `bundle exec jekyll serve --livereload`.

## Deploying

Push to the `main` branch of the `TanoojMadhuvan.github.io` repo — GitHub
Pages builds and deploys it automatically (this repo only uses plugins on
GitHub Pages' supported list: `jekyll-feed`, `jekyll-seo-tag`,
`jekyll-sitemap`, so no GitHub Actions workflow is needed).

## Before going live

- [ ] Write the homepage intro paragraph (`index.md`)
- [ ] Fill in the highlight bullets and stat-strip values in `index.md`'s
      front matter (`highlights:` and `stats:`)
- [ ] Set `linkedin_username` in `_config.yml`
- [ ] Fill in `about.md` and `resume.md`
- [ ] Drop your real `resume.pdf` in `assets/files/`
- [x] Confirm the repo link in `foundation.md` — `github.com/tanooj-comp-arch/foundation-core`
- [ ] Replace each track's `REPLACE_ME` intro/why text in `_tracks/`
- [ ] Replace or delete the placeholder posts in `_posts/`
