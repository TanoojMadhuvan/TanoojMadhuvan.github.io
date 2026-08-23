source "https://rubygems.org"

# Pins Jekyll + plugin versions to whatever GitHub Pages currently runs in
# production, so `bundle exec jekyll serve` locally matches the deployed site.
gem "github-pages", group: :jekyll_plugins

group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
end

# Windows/JRuby: tzinfo data isn't bundled with the OS
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby]
# NOTE: deliberately not depending on the `wdm` gem for Windows file-watching —
# it's unmaintained and fails to compile against modern Ruby headers. Without
# it, the Listen gem (used by `jekyll serve --livereload`) falls back to
# polling, which works fine for local preview.

# Ruby 3.0+ removed webrick from the standard library; jekyll serve needs it
gem "webrick", "~> 1.8"
