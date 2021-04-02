# alexandraulsh.com

https://www.alexandraulsh.com/

My personal website.

## Dependencies

This websites uses the same versions of Ruby (v2.7.1) and Jekyll (v3.9.0) as [GitHub Pages](https://pages.github.com/versions/).

Install rbenv with [homebrew](https://brew.sh/) and optionally set Ruby v2.7.1 as the default:

```sh
brew install rbenv
rbenv install 2.7.1
rbenv global 2.7.1
```

Set up [rbenv in your shell](https://github.com/rbenv/rbenv#how-rbenv-hooks-into-your-shell):

```sh
echo 'eval "$(rbenv init -)"' >> ~/.zshrc
source ~/.zshrc
```

Upgrade bundler to v2.2.8:

```sh
gem install bundler:2.2.8
```

Install Jekyll v3.9.0:

```sh
gem install jekyll:3.9.0
```

## Install locally

After installing dependencies, to install this website locally:

```sh
git clone git@github.com:alulsh/alulsh.github.io.git
cd alulsh.github.io
bundle install
```

## Run locally

Run `jekyll serve --watch` then visit http://127.0.0.1:4000/ in a web browser.