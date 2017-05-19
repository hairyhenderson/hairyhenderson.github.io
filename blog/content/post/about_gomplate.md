+++
categories = ["tools", "open-source", "docker", "golang", "gomplate"]
date = "2017-05-18T23:14:49-04:00"
description = "An introduction to a project I've been working on for a long while..."
keywords = ["gomplate"]
title = "About gomplate"
+++

I released `gomplate` version 1.6.0 ([here][gomplate-releases]) a couple weeks ago.
It was a fairly big release, as these things go, but I realized that I haven't ever
really introduced `gomplate` formally. So this post is an attempt to describe what
it is, why I thought it needed to exist, and what's happened with it over the
last year and a half.

## And now, some history...

I'm a big fan of [Docker][], and I've been using it for a few years, and trying
to wrap most programs I use as Docker images. When you're dockerizing _all the
things_, you start needing to write a lot of shims and glue for programs that
weren't _really_ designed to be run in a container in the first place.

So as I'm sure most people have done in this situation, I began writing lots of
shell scripts. I wanted to mostly stick to [12-factor][] principles, and so
there was lots of passing config through environment variables, and then mutating
config files to apply those variables.

To feed variables in, I generally used [`envsubst`][], which is pretty much
perfect for taking a file containing something like `name=${NAME}`, and turning
it into `name=foo`. Things went _pretty_ well until I started encountering files
where shell-style variables had special meaning (like, for example, in shell
scripts). I resorted to some pretty nasty `sed` scripts for a short time, but I
hoped there would be something better out there.

Meanwhile, I had been playing around with [Go][], mostly helping out with the
[Docker Machine][] project, and one feature that I really like about the
language was the [`text/template`][] package. Essentially it's a templating
language built in to the Go standard library. An example of how `docker-machine`
uses it is:

```console
$ docker-machine ls --format '{{ .Name }} is running {{ .DockerVersion }}'
gallium is running v17.05.0-ce
mercury is running v17.04.0-ce
tin is running v17.05.0-ce
```

_(the template is the text passed to `--format`)_

I really liked the format, and so I set out to find a commandline tool that could
render templates using `text/template`. I couldn't find any obvious candidates
(though I've since discovered that alternatives _did_ exist), so I decided to go
off and write my own...

### Version 0.0.1

I spent a few hours one cold Saturday morning, and came up with [v0.0.1][].
At the time, I named it `templater` for lack of a better name. I messaged my
friend and colleague [Drew MacInnis](https://twitter.com/drmdrew), and told him
about it - he forked it almost immediately and started playing with it, and then
we started talking about names:

> DH: can you think of a better name for it?

> DM: No 💡 yet... `goenvsubst` seems bland...

> DH: yeah... `gontemplation` was the best I could come up with so far, but it's a bit contrived

> DM: `gollumplater`... yes, precious 😉

> DH: lol

> DM: `gomplate`?

> DH: Nice. Sold.

My thanks certainly goes out to Drew for what I think is an _awesome_ name!

### The early versions...

Over the next few months I started to add functionality, in the form of built-in
functions. One of the coolest things about Go templates is that you can attach
pretty much any function you want, and then use it in the template.

One big thing that happened early on was `gomplate` started becoming cloud-aware.
It gained the ability to interface with AWS's APIs, so that I could include the
name of the EC2 region where `gomplate` was running. This meant that I didn't
have to tell it explicitly which region it was in (and risk getting it wrong
-- _something that's happened more times than I'd care to admit..._).

A pile of other EC2-related functions were added, and then some string-manipulation
functions, including the ability to parse JSON strings and deal with lists.

### Datasources!

In the spring of 2016, I decided that `gomplate` needed to learn how to read
data from places other than environment variables and cloud APIs.

At first I really just wanted the ability to read a JSON file from the local
filesystem, but it became clear that I'd need to support remote files too at
some point, so I chose to use URIs as the generic interface.

Long story short, the new `datasource` function and `--datasource` commandline
argument landed in [v0.5.0][].

### To 1.0 and beyond

In July, I decided to finally declare that `gomplate` was "complete" (ha!) and
released version 1.0.0:

{{< tweet 753434572604317696 >}}

Obviously it _wasn't_ complete...

### Vault support

One nifty feature introduced in v1.2.0 is support for using HashiCorp's [Vault][]
as a datasource. This means that `gomplate` can be used to safely inject secrets
into config files, among other things.

As an aside, an interesting pattern that has emerged is to use `gomplate`
instead of the `vault` client, something like this:

```bash
eval $(echo '
    export A_SECRET={{(datasource "vault" "secret1").value}};
    export ANOTHER_SECRET={{(datasource "vault" "secret2").value}};
  ' | gomplate -d vault=vault:///secret/mysecrets/)
echo "the secrets are $A_SECRET and $ANOTHER_SECRET"
```

While it's a _bit_ more clunky than the equivalent `vault read` calls would be,
it allows direct access to more complex objects stored in Vault, and the binary
is smaller (`vault` is ~50MB vs the slim `gomplate` build around ~3.5MB) -- this
is important when building and shipping Docker images around the Internet!

### Packaging `gomplate`

Along the way, it became clear that pulling binaries from
[GitHub's releases page][gomplate-releases] isn't always ideal. There's a reason
that package managers exist!

And so, around the time I released 1.2.0, I added a Homebrew "tap" for use on
macOS, so it can be installed with just:

```console
$ brew tap hairyhenderson/tap
$ brew install gomplate
```

Maybe when `gomplate` _really_ hits the mainstream, I won't have to maintain
a separate tap 😉.

Later on, I [packaged](https://github.com/alpinelinux/aports/blob/master/community/gomplate/APKBUILD)
`gomplate` for [Alpine Linux][], which is a _fantastically small_ Linux distro
which is heavily used as a base for Docker containers. Since `gomplate` is quite
often used inside a container, it only made sense!

And, while I'm talking about Docker, I probably shouldn't forget to mention that
there _is_ a [`gomplate` image][] on DockerHub.

### Always keep releasing...

Since then, there have been a number of new functions added, some small
features (like the ability to choose template delimiters other than the
default `{{`/`}}`), and a number of performance improvements.

Along the way, a number of excellent people joined the effort (8, to be exact,
[not counting a bot](https://github.com/hairyhenderson/gomplate/graphs/contributors)),
and some really useful features landed. And so I got around to releasing 1.6.0.

The main highlights in 1.6.0 are:

- the ability to use authenticated HTTP/HTTPS datasources (`--datasource-header`/`-H`
  argument)
- ability to configure Vault datasources with files, and avoiding passing secrets
  through environment variables (allowing use of Docker Swarm Secrets)
- the ability to render multiple templates at once, and whole directories of
  templates (contributed by a very patient [@rhuss](https://github.com/rhuss)).
- some _real_ integration tests run on each build, (_finally_!)

The biggest highlight for me though has been the involvement of more and more
new users and contributors; it's _incredibly_ satisfying to see a community start
to form, and to know that people are getting value out of something I work on in
my spare time.

### The future!

I hope the future holds _at least_ a 1.7.0 version - at the time I'm writing
this, it's been almost 20 days since 1.6.0 and there are
[at least 15](https://github.com/hairyhenderson/gomplate/compare/v1.6.0...master)
changes since then...

Broadly, though, I'm hoping to do _more_ with `gomplate`:
- more (and better) documentation
- more packages for linux distros and other OSes
- more support for cloud providers (that aren't AWS)!

Finally, if you've made it this far without getting bored, thank you! And if you
haven't yet, go ahead and throw a ⭐️ at https://github.com/hairyhenderson/gomplate!

<iframe src="https://ghbtns.com/github-btn.html?user=hairyhenderson&repo=gomplate&type=star&count=true" frameborder="0" scrolling="0" width="170px" height="20px"></iframe>

[gomplate-releases]: https://github.com/hairyhenderson/gomplate/releases
[Docker]: https://docker.com
[12-factor]: https://12factor.net
[`envsubst`]: https://linux.die.net/man/1/envsubst
[Go]: https://golang.org
[Docker Machine]: https://github.com/docker/machine
[`text/template`]: https://golang.org/pkg/text/template
[v0.0.1]: https://github.com/hairyhenderson/gomplate/releases/tag/v0.0.1
[v0.5.0]: https://github.com/hairyhenderson/gomplate/releases/tag/v0.5.0
[Vault]: https://vaultproject.io
[Alpine Linux]: https://alpinelinux.org
[`gomplate` image]: https://hub.docker.com/r/hairyhenderson/gomplate
