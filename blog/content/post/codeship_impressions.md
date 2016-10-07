+++
categories = [ "notes", "reviews", "docker" ]
date = "2016-09-27T20:02:08-04:00"
description = "Some first impressions trying out Codeship for a Docker workflow"
keywords = [ "ci", "docker", "review" ]
title = "Codeship first impressions"

+++

first impressions:

- URL to the project is opaque (https://app.codeship.com/projects/176008 vs something with the repo in the URL like https://circleci.com/gh/hairyhenderson/jiraprinter)
- Docker support only available (for free) with a 14-day trial, even for OSS projects?
- weird:
  ```
  preparing...
  Node Version read from package.json: 0.10
  Node Version used: 0.10
  ```
  - I didn't specify an engine version at all, and this is a pretty crappy default! (should default to latest LTS)
  - seems like I ended up with 2 versions of Node (second installed by `nvm`)
- seems fast!
- weird test failure at first - maybe the filesystem isn't writeable:
  ```
  Error: EACCES: permission denied, open '/mocha.xml'
  ```
  - this was my bad - my `Makefile` had a CircleCI-specific env var

- free t-shirt!

- unclear what happens to test results... seems not to be support for rendering xUnit-format results?
-
