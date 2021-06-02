source 'https://rubygems.org'

ruby '2.7.2'

require 'json'
require 'open-uri'
versions = JSON.parse(::URI.parse('https://pages.github.com/versions.json').read)

gem 'github-pages', versions['github-pages']

gem 'jekyll'

# github-pages depends on ffi, but I get an error trying to install more recent versions :(
gem 'ffi', '1.11.1'

gem 'jekyll-paginate'
gem 'jekyll-theme-slate'
