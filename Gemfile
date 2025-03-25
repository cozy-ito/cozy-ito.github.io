# frozen_string_literal: true

source "https://rubygems.org"

# Jekyll 직접 지정
gem "jekyll", "~> 4.3.0"

# Chirpy 테마 직접 지정 (gemspec 대신)
gem "jekyll-theme-chirpy", "~> 6.0"

# 필요한 Jekyll 플러그인
group :jekyll_plugins do
  gem "jekyll-archives"
  gem "jekyll-sitemap"
  gem "jekyll-paginate"
  gem "jekyll-redirect-from"
  gem "jekyll-seo-tag"
end

gem "html-proofer", "~> 5.0", group: :test

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]

# 성능 향상을 위한 HTTP 서버
gem "webrick", "~> 1.8"