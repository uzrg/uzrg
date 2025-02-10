# frozen_string_literal: true

source "https://rubygems.org"

# Remove gemspec unless you have a specific .gemspec file for your theme
# (Most Chirpy users don't need this)
# gemspec

gem "jekyll", "~> 4.3.0"  # Updated to match Chirpy's requirements

# Remove github-pages gem - it conflicts with custom themes
# group :jekyll_plugins do
#   gem "github-pages"
# end

# Specify Chirpy theme directly from GitHub
gem "jekyll-theme-chirpy", git: "https://github.com/cotes2020/jekyll-theme-chirpy.git"

# Add required plugins explicitly
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-seo-tag"
  gem "jekyll-archives"
  gem "jekyll-sitemap"
end

group :test do
  gem "html-proofer", "~> 4.4"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.1.1", platforms: [:mingw, :x64_mingw, :mswin]

gem "http_parser.rb", "~> 0.6.0", platforms: [:jruby]
