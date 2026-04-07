source "https://rubygems.org"

# Use plain Jekyll for local dev (github-pages forces safe:true which blocks
# symlinked _includes — the submodule symlinks work fine on GitHub Pages itself).
gem "jekyll", "~> 3.10"

# Required for `bundle exec jekyll serve` on Ruby 3+
gem "webrick"

# Plugins explicitly used in _config.yml
group :jekyll_plugins do
  gem "jekyll-seo-tag"
  gem "jekyll-sitemap"
  gem "jekyll-feed"
  gem "jekyll-paginate"
  gem "jekyll-include-cache"
  gem "kramdown-parser-gfm"
end
