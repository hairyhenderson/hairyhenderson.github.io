+++
categories = [ "tools" ]
date = "2015-12-12T13:21:28-05:00"
description = ""
keywords = []
title = "asciinema"

+++

Ran into this a while back, but I keep forgetting about it:  https://asciinema.org

## Using

```console
$ asciinema rec
...
$ exit
...
```

It'll start a new shell and record all the things (including timing and mistakes!), then upload it to their site for serving.

## Modifying

Download it:

```console
$ curl -sL https://asciinema.org/a/abcd1234.json > rec.json
```

_Edit the file..._

Upload it:

```console
$ asciinema upload rec.json
```

## Other useful options

Record something that isn't a shell:

```console
$ asciinema rec -c node
~ Asciicast recording started.
~ Hit Ctrl-D or type "exit" to finish.
> console.log('hello world')
hello world
undefined
> ~ Asciicast recording finished.
~ Press <Enter> to upload, <Ctrl-C> to cancel.
```

_fin_
