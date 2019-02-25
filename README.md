# alexandraulsh.com

https://www.alexandraulsh.com/

My personal website.

## Prerequisites

You'll need Ruby with [Bundler](https://bundler.io/). Use the version of Ruby installed with [homebrew](https://brew.sh/), not the default Ruby on macOS.

To use the homebrew version of Ruby:

```
brew install ruby
echo 'export PATH=/usr/local/opt/ruby/bin:$PATH' >> ~/.bash_profile
source ~/.bash_profile
```

Then install Bundler with `gem install bundler`.

## Setup

```sh
git clone git@github.com:alulsh/alulsh.github.io.git
cd alulsh.github.io
bundle install
```

You may need to run `bundle update --bundler`.

## Run locally

Run `bundle exec jekyll serve --watch` then visit http://127.0.0.1:4000/ in a web browser.