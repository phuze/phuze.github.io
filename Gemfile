source "https://rubygems.org"

# Use github pages, instead of the core jekyll
gem "github-pages", group: :jekyll_plugins

# Plugins
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-paginate"
  gem "jekyll-seo-tag"
  gem "jemoji"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", ">= 0.1.1" if Gem.win_platform?

# Enables live-reloading in windows
# https://rubygems.org/gems/webrick/versions/1.3.1
# https://github.com/jekyll/jekyll/issues/8926
gem "webrick"