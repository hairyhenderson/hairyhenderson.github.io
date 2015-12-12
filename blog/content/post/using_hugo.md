+++
categories = [ "notes" ]
date = "2015-12-12T12:38:40-05:00"
description = ""
keywords = []
title = "Using Hugo"

+++

Hugo's pretty simple to use, but I use it so sporadically I figured I should jot down how I use it...

_**Note:** All the following commands assume I'm in the `/blog` subdirectory of the
[repo](https://github.com/hairyhenderson/hairyhenderson.github.io)._

## Installation

I installed it with Homebrew:

```console
$ brew install hugo
```

so to upgrade it I just `brew update && brew upgrade`.

## Previewing

To get `hugo` to render on-the-fly:

```console
$ hugo server --watch --buildDrafts
```

## Writing new articles

To start a new post:

```console
$ hugo new post/title_of_post.md
```

Or if it's a static page (about/etc)

```console
$ hugo new title_of_page.md
```

Edit the page:

```console
$ atom content/<path to .md file>
```

Update the `categories` and `title` fields, and set `draft = true` in the front matter if necessary.

## Publishing

Regenerate the site

```console
$ hugo
```

Then `git add`/`git commit`/`git push` to taste.

_fin_
