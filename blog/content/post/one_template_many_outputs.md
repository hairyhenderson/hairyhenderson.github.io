---
categories: [ tools, open-source, docker, golang, gomplate ]
date: "2019-03-04T22:20:44-05:00"
description: |
  In gomplate v3.3.0, you can now use `file.Write` and `tmpl.Exec` to render
  multiple output files from a single input template. _Read on to find out how..._
keywords: [ gomplate ]
title: One template, many outputs
---

## TL;DR

In [gomplate v3.3.0][], you can now use [`file.Write`][] and [`tmpl.Exec`][] to render multiple output files from a single input template. _Read on to find out how..._

## Background

When I first wrote [gomplate][], I started with the usual _filter_ approach; a single input file (the template) was processed to produce a single output:

```console
$ gomplate < input.tmpl > output.txt
```

This approach works well, but quite a lot can happen in a template, and when you need to produce many outputs, calling `gomplate` multiple times can be cumbersome.

So in [gomplate v1.6.0][], new support was added for processing directories of files:

```console
$ gomplate --input-dir ./templates/ --output-dir ./out/
```

This is great for the case where the same number of outputs as inputs are desired, but another use-case that's come up a few times is being able to produce multiple outputs from a single input template.

Consider a template like the following example:

```go
{{ range $region := slice "AP" "EU" "US" -}}
Herein lies the {{ $region }} configuration...
{{ end -}}
```

If you render this with gomplate, you'll get a single output with the "configuration" for each region contained within:

```console
$ gomplate -f in
Herein lies the AP configuration...
Herein lies the EU configuration...
Herein lies the US configuration...
```

Obviously this is useful in lots of situations, and I personally use gomplate like this to create similarly repetitive configurations quite regularly.

But what about the case where each region/environment/whatever needs a differently-named configuration file? Before [v3.3.0][gomplate v3.3.0], the only way to accomplish this was to run `gomplate` multiple times, with different inputs.

## Composing a solution

This has come up a few times in my own work, and when one of gomplate's users recently [asked](https://github.com/hairyhenderson/gomplate/issues/485) how to do this, I knew it was time to make it work!

Now, the solution that I came up with is _not_ necessarily the simplest, but I think it's quite flexible, and will enable other use-cases I haven't thought of. Instead of building a specific multi-output feature directly into gomplate, I decided to add two new functions, and address [another issue](https://github.com/hairyhenderson/gomplate/issues/444) at the same time.

The basic idea revolves around using a [nested template][] and a loop that decides what to name output files, and writes them. Let's look at an example:

```go
{{ define "mytemplate" }}
Herein lies the {{ .region }} configuration...
{{ end -}}

{{- range $region := slice "AP" "EU" "US" -}}
{{- $ctx := dict "region" $region }}
{{- $outPath := printf "config_%s.txt" $region }}
{{- tmpl.Exec "mytemplate" $ctx | file.Write $outPath }}
{{- end -}}
```

When we run this, we can now see 3 output files:

```console
$ gomplate -f in
$ ls
config_AP.txt  config_EU.txt  config_US.txt  in
```

To separate concerns a bit more, we can use [external nested templates][] to split the template file into two: the actual template to render, and the loop to control output:

_in:_
```go
Herein lies the {{ .region }} configuration...
```

_render:_
```go
{{- range $region := slice "AP" "EU" "US" -}}
{{- $ctx := dict "region" $region }}
{{- $outPath := printf "config_%s.txt" $region }}
{{- tmpl.Exec "mytemplate" $ctx | file.Write $outPath }}
{{- end -}}
```

```console
$ gomplate -t mytemplate=in -f render
$ ls
config_AP.txt  config_EU.txt  config_US.txt  in  render
```

Same result, with cleaner input!

### Explanation

The `render` template has effectively 3 parts:

1. some sort of loop or loops to pick which data will go in which files
  - this is the `range` loop, using a simple [`slice`][] for input
  - more complex use-cases would use a [`datasource`][] instead
2. some code to create a unique filename
  - in the above example, `printf` is used to produce a filename that contains the region
  - you can also use the [`filepath`][] functions to build more complex paths, including directories
3. `tmpl.Exec` to render the template with a given context, piping to `file.Write` to write to a file
  - building the context (`$ctx` above) is made slightly easier with the use of [`dict`][]

## Summary

Of course, this can all be accomplished with a mess of shell scripts that call `gomplate` multiple times, but besides keeping things a bit cleaner, this approach can lead to huge performance gains. I'll let [@rayjlinden][], the author of the issue that inspired all this, provide the proof:

> Wow. This is really cool. It makes my Dockerfile generation an order of magnitude faster. (I'm generating ~70 files in the time it previously took to generate about 3. Nice!

_[#485 (comment)](https://github.com/hairyhenderson/gomplate/issues/485#issuecomment-464371492)_

🏎🎉

[gomplate]: https://github.com/hairyhenderson/gomplate/
[gomplate v1.6.0]: https://github.com/hairyhenderson/gomplate/releases/tag/v1.6.0
[gomplate v3.3.0]: https://github.com/hairyhenderson/gomplate/releases/tag/v3.3.0
[`file.Write`]: https://docs.gomplate.ca/functions/file/#filewrite
[`tmpl.Exec`]: https://docs.gomplate.ca/functions/tmpl/#tmplexec
[`range`]: https://golang.org/pkg/text/template/#hdr-Actions
[nested template]: https://docs.gomplate.ca/syntax/#nested-templates
[external nested templates]: https://docs.gomplate.ca/syntax/#external-templates
[`slice`]: https://docs.gomplate.ca/functions/coll/#collslice-_deprecated_
[`datasource`]: https://docs.gomplate.ca/datasources/
[`filepath`]: https://docs.gomplate.ca/functions/filepath/
[`dict`]: https://docs.gomplate.ca/functions/coll/#colldict
[@rayjlinden]: https://github.com/rayjlinden
