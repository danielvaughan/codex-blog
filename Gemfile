source "https://rubygems.org"

# GitHub Pages — pinned gem set used by github.io builds. Locks Jekyll
# version + plugin set and runs in safe mode (which is what production
# GitHub Pages uses). We are safe-mode-compatible: no symlinks anywhere
# in the source tree, sass load_paths only point to subdirectories within
# the site source root.
gem "github-pages", group: :jekyll_plugins

# Git-based last_modified_at for SEO freshness signals.
# Not in the github-pages whitelist, so requires GitHub Actions deployment.
gem "jekyll-last-modified-at", group: :jekyll_plugins

# Required for `bundle exec jekyll serve` on Ruby 3+
gem "webrick"
