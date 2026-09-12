# static-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

Static files over HTTP, done properly: a path resolver that cannot
escape its root, ETags and conditional requests, byte ranges,
precompressed `.gz` and `.br` siblings, directory index policy, and
cache-control presets.

It is what you reach for when your program has a directory of files and
a port.

It is a port of [tower-http](https://github.com/tower-rs/tower-http)'s
`ServeDir` and [whitenoise](https://whitenoise.readthedocs.io).  Six
modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **server** | `staticfs` | you are mounting a directory |
| the **paths** | `staticpath` | you are auditing what can be reached |
| the **validators** | `staticmeta` | you are wondering why a browser re-downloads |
| the **ranges** | `staticrange` | a video will not scrub, or a download will not resume |
| the **encodings** | `staticenc` | you have `.gz` siblings and want them used |
| the **refusals** | `staticerr` | you are reading a 404 in a log |

## Adding it, and checking it

```bash
novo pkg add static-nv           # into your novo.toml
novo pkg build                   # type- and effect-check the package
novo test --isolate tests/staticpath_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: static-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use staticfs

fn main() [io, fs, net, time, mutate, async]
    let root = staticfs.root("/srv/site")
    match staticfs.scan(root)
        Err(_) => println("that root is not readable")
        Ok(index) =>
            let app = (req) =>
                match staticfs.serve(index, req.path, req.query,
                                     (n) => http.server.request_header(req, n) ?? "",
                                     req.method == "HEAD")
                    Err(_) => http.server.text(500, "500\n")
                    Ok(s)  => response_from(s)
            match http.server.serve_pool(8080, app, http.server.serve_opts(), 16)
                Some(_) => println("bind failed")
                None    => println("shut down cleanly")
```

`scan` walks the root once at start-up.  Every request after that is a
lookup plus one positional read of the bytes actually asked for.

## The load-bearing interface

`staticpath.resolve` — and the fact that it is `[]`.

```novo norun:pseudo
pub fn resolve(target: Str, p: StaticPolicy,
               exists: fn(Str) -> Int []) -> StaticResolved []
```

Path traversal is the oldest vulnerability a file server has, and the
reason it keeps happening is not that the rules are hard.  It is that
the rules are usually checked **against a filesystem**: a suite covers
the ten inputs somebody thought of, the eleventh is a CVE, and nobody
can read the rule set without running it.

So the resolver here performs nothing.  The `exists` argument is a
predicate the caller supplies — `staticfs` passes one backed by the
filesystem, a test passes one backed by a list of three names — and
everything else is arithmetic over a string.  The consequence is that
[`tests/staticpath_tests.nv`](tests/staticpath_tests.nv) **is** the
rule set: every refusal, with the input that triggers it, readable end
to end in one screen.  That file is what a reviewer audits.

### The order of the two steps is the whole argument

1. **Split** on the literal `/` bytes.
2. **Percent-decode** each segment.
3. **Check** each decoded segment.

Decoding before splitting is the bug.  `/files/%2e%2e/%2e%2e/etc/passwd`
has no `..` in it as it arrives; a check run first sees `%2e%2e`,
passes it, and the decode afterwards produces exactly the traversal the
check was for.

Splitting after decoding is the other bug, and it is subtler.
`/files/a%2Fb` is **one** segment whose name contains a slash.  A
resolver that decoded first would see two segments and hand back a path
one directory deeper than the request named.  So the split runs on the
literal bytes and a segment that decodes to something containing a
separator is **refused** — not accepted, not re-split.

This is the same argument
[router-nv](https://github.com/novolang/router-nv) makes about matching
before decoding, arrived at from the other end.

### What `[]` cannot buy: symbolic links

The resolver proves the path is inside the root **as a string**.  It
cannot prove that `root/uploads/x` is not a symbolic link to
`/etc/shadow`, because that question belongs to the filesystem.

Neither can `staticfs`, and the reason is worth stating plainly: **the
standard library answers no symbolic-link question at all** — there is
no `realpath`, no `readlink`, and no link flag on any metadata call.
So `staticfs.containment_is_lexical_only()` is published, it answers
`true` today, and a deployment's own audit can assert it.  A root
written by a build is safe; a root that accepts uploads and follows
links is not.  The gap is filed against the standard library rather
than papered over, and the day a `realpath` lands, that function's
answer changes and nothing else in the surface does.

## What it replaces

The standard library's `http.server.url_to_fs_path` is thirteen lines
and its containment check is `str.contains(target, "..")`.  That check
is wrong in both directions:

- It **passes** `%2e%2e`, `%2F`, a NUL byte and a backslash.
- It **refuses** `a..b.txt`, which is a real file name that appears in
  archives and in generated documentation.

And around it there is no ETag, no `If-None-Match`, no `Range`, no
`Accept-Encoding`, no `Cache-Control`, and `fs.read_bytes` reading a
whole file into memory to serve any part of it.

## The layer, and why

`host`.  Something has to read the disk.

**Five of the six modules are `[]`.**  `staticerr`, `staticpath`,
`staticmeta`, `staticrange` and `staticenc` touch nothing: a path is
arithmetic over a string, a conditional request is a comparison of two
strings, a `Range` header is integers, and an `Accept-Encoding` is
tokens with weights.  Only `staticfs` reads, and it declares `[fs]` —
no `[io]`, because a library does not print, and no `[net]`, because
this package answers a **value** and the caller's server writes it.

## Three details that cost real money when they are wrong

**`Vary: Accept-Encoding` is not optional.**  A shared cache that
stored a gzip response under the URL alone serves those bytes to the
next client — including one that cannot decode them.  The result is a
stylesheet rendering as binary for everybody behind that cache until it
expires.  `staticenc.vary_header` is emitted on **every** response,
including the uncompressed ones, because the header describes the URL
and not the response that happened to be chosen.

**`If-None-Match` overrides `If-Modified-Since`.**  RFC 9110 § 13.2.2
says so and implementations skip it.  A browser revalidating sends both
headers; a server that checks the date independently answers 200 with a
whole file to a request that deserved a 304 — on every revalidation,
forever.  `staticmeta.evaluate` is the whole precedence table in one
function for exactly this reason.

**A multi-range request is an amplifier.**  `bytes=0-0,1-1,2-2,…` with
ten thousand ranges produces a multipart response many times the size
of the file, and `bytes=0-999999,0-999999,0-999999` is three copies of
it.  `staticrange.StaticRangeLimits` is a ceiling on both, checked at
parse, rather than a paragraph in a document.

## The ETag is a content hash, and that was not a free choice

Every other static file server computes a **weak** validator from the
file's size and its modification time: one stat call, right almost
always, costs nothing.

This package cannot.  The standard library answers no modification time
— `std.fs` has `size`, `exists`, `is_file`, `read_at` and no stat, and
`std.path` has no metadata either — so size-and-mtime is not a
validator that can be computed here at all.

So the validator is `sha256` of the content, which is a **strong** one:
correct by construction, usable for a range request's `If-Range` (a
weak one is not, § 13.1.5), and expensive on a large file.  That
expense is why `staticfs.scan` exists: the hash is computed once per
file at start-up, and every request after that is a comparison.  For
the deployment that cannot afford the scan,
`staticmeta.entry_with_etag` takes a validator the **build** already
computed — which is a shape a static site generator already has.

`Last-Modified` has the same root and a different consequence: with no
modification time there is nothing truthful to put in it, so this
package emits it only from a time the caller supplied, and leaves it
absent otherwise.  That costs nothing — `ETag` and `If-None-Match` do
the whole job, and § 13.1.3 makes `If-Modified-Since` the weaker of the
two anyway.  Both gaps are filed rather than worked around.

## It does not compress

flate-nv and brotli-nv are **named** here and are not dependencies.
This package serves the `.gz` or `.br` sibling a build already
produced, as bytes, with `Content-Encoding` set.

Compressing per request is the wrong place: the same file is compressed
once per requester rather than once per deploy, at a level nobody can
budget for.  `staticenc.worth_compressing` is published so the **build**
can use the same rule the server does — which is what stops a
deployment shipping ten thousand useless `.gz` files beside its PNGs.

## Four headers with no home yet

`Range`, `If-None-Match`, `If-Modified-Since` and `Accept-Encoding` are
parsed here because nothing else on the registry parses them.
http-codec-nv reads a request line, the header fields as name-value
pairs, and the body's framing, and stops.  mime-nv parses `Accept`, and
cannot parse `Accept-Encoding`: RFC 9110 § 12.5.3's grammar is a list of
**codings** with q-values where § 12.5.1's is a list of **media-ranges**
with q-values — the weights are the same and the tokens are not, since
`gzip` has no `/` in it.

A second consumer — a caching proxy, an HTTP client that revalidates
its own cache — would be the reason to move those four into a shared
package.  Until there is one, they live here and this paragraph is the
record of the decision.

## What `orbit/website` would take

All of it, and the change is small: the registry's site serves a
generated directory today through the standard library's thirteen-line
handler, with no ETag, no conditional requests, no ranges and no
precompressed siblings, reading every file whole into memory to answer
any part of it.

Swapping it for `staticfs.scan` at start-up and `staticfs.serve` per
request would give the documentation pages 304s instead of full
re-downloads, serve the `.gz` the generator can already write, and read
a byte range of the wasm blob rather than all of it.  The one thing to
decide first is the cache policy: the site's fingerprinted assets want
`staticmeta.fingerprinted`, and its HTML pages want
`staticmeta.revalidated`, and using one for both is the mistake the
presets exist to prevent.

## Related

- [mime-nv](https://github.com/novolang/mime-nv) — `Content-Type`, and
  the sniffing rule a static file server is the one program that can
  set `nosniff` for
- [router-nv](https://github.com/novolang/router-nv) — the same
  don't-decode-first argument, from the other end
- [session-nv](https://github.com/novolang/session-nv),
  [oauth2-nv](https://github.com/novolang/oauth2-nv) — the rest of a
  web tier's host half
