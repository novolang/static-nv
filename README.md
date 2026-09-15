# static-nv

A static file server answers an HTTP request by sending a file from a
directory. Doing it correctly means more than reading the file: the
request's path must not be able to escape that directory, a browser
that already has the file must be told so rather than sent it again,
and a client asking for part of a file must get that part. The rules
are in
[RFC 9110](https://www.rfc-editor.org/rfc/rfc9110), sections 13 and 14.
This package implements them. It is a port of
[tower-http](https://github.com/tower-rs/tower-http)'s `ServeDir` and
of [WhiteNoise](https://whitenoise.readthedocs.io).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What serving a file involves

The **root** is the directory a server publishes. A request's target is
turned into a path relative to it, and **path traversal** is the attack
in which a crafted target names a file outside it.

A **validator** is a short value standing for a file's current
contents. RFC 9110 section 8.8.3 calls it an **entity tag**, sent in the
`ETag` header. A **strong** validator changes whenever the bytes do. A
**weak** one, written `W/"..."`, means only that two representations
are equivalent for caching.

A **conditional request** carries a validator the client already holds.
`If-None-Match` asks the server to answer 304 Not Modified if the
client's copy is current. `If-Modified-Since` asks the same question
against a date. `If-Range` asks for a byte range only if nothing has
changed (section 13.1.5).

A **range request** asks for part of a file, with the `Range` header
(section 14.2). A server that supports them answers 206 Partial
Content, sends `Content-Range`, and advertises `Accept-Ranges: bytes`.
A request for several ranges at once is answered as a multipart body.

A **precompressed sibling** is a file such as `style.css.gz` sitting
beside `style.css`, produced by the build. A client that sends
`Accept-Encoding: gzip` can be sent that file's bytes with
`Content-Encoding: gzip` set, without anything being compressed at
request time.

`Cache-Control` (section 5.2 of RFC 9111) tells a cache how long a
response may be reused. `Vary` tells a cache which request headers the
chosen response depended on.

Five of this package's six modules perform no input or output. Only
`staticfs` reads the disk, and it declares `[fs]` alone: a library does
not print, and this package answers a value rather than writing to a
socket.

## Install

```
novo pkg add static-nv
```

## Example

```novo
use staticerr
use staticmeta
use staticpath
use staticrange

// What the resolver asks about each path. `staticfs.kind_of` is the
// filesystem's answer; this one is a directory with a single file in it.
fn tiny_fs(relative: Str) -> Int
    if relative == "style.css"
        staticpath.kind_file()
    else
        staticpath.kind_missing()

fn main() [io]
    // A target with the traversal spelled in percent-encoding, so no
    // `..` is visible in it as it arrives.
    match staticpath.check("/%2e%2e/%2e%2e/etc/passwd", staticpath.defaults())
        Ok(rel) => println("would serve ${rel}")
        Err(r)  => println("${staticerr.refusal_status(r)}: ${staticerr.refusal_detail(r)}")

    // An ordinary target, resolved against the same policy.
    println(staticpath.relative_of(staticpath.resolve("/style.css", staticpath.defaults(), tiny_fs)))

    // The validator for that file's bytes, and the header a cache reads.
    let e = staticmeta.entry_with_etag("style.css", 1024, "abc123", false)
    println("ETag: ${e.etag}")
    println("Cache-Control: ${staticmeta.cache_header(staticmeta.fingerprinted())}")

    // The first thousand bytes of it, as a `Content-Range`.
    match staticrange.resolve_closed(0, 999, e.size)
        Some(r) => println("Content-Range: ${staticrange.content_range(r, e.size)}")
        None    => println(staticrange.unsatisfiable_header(e.size))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: static-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `staticerr` | Every refusal, the status each produces, and whether a refusal looks like somebody probing. |
| `staticpath` | A request target turned into a path inside the root, the policy that decides what is servable, and the checks on one segment. |
| `staticmeta` | Entity tags, HTTP dates, the conditional-request precedence table, and the cache-control presets. |
| `staticrange` | The `Range` header parsed and bounded, and the `Content-Range` and multipart headers a 206 needs. |
| `staticenc` | `Accept-Encoding` parsed, the precompressed sibling chosen, and the `Vary` header. |
| `staticfs` | The only module that reads a disk: the root, the index built at start-up, and the two serve calls. |

## How to choose an entry point

**`staticfs.scan` at start-up and `staticfs.serve` per request** is the
ordinary path. `scan` walks the root once, computing each file's
validator, and every request after that is a lookup plus one positional
read of the bytes actually asked for.

**`staticfs.serve_live` reads metadata per request**, for a root whose
files change while the server runs.

**`staticpath.resolve` and `check` are the resolver on its own.**
`resolve` answers what kind of thing the target names; `check` answers
only the relative path or the refusal. Both take a predicate rather
than touching a disk.

**`staticmeta.evaluate` is the conditional-request answer**, as a
status code. `staticmeta.conditions_of` reads the request's headers
into the value it takes.

**`staticrange.parse` reads a `Range` header** against a file size and
the caller's limits. The three `resolve_*` functions handle one range
each, for a caller parsing the header itself.

## The rules a user needs

1. **Split the target on literal `/`, then percent-decode each
   segment, then check each segment.** That order is the whole of the
   resolver's correctness.
2. **Decoding before checking is the bug.**
   `/files/%2e%2e/%2e%2e/etc/passwd` contains no `..` as it arrives. A
   check that runs first passes it, and the decode afterwards produces
   the traversal the check was for.
3. **Splitting after decoding is the other bug.** `/files/a%2Fb` is one
   segment whose name contains a slash. A resolver that decoded first
   would see two segments and answer a path one directory deeper than
   the request named. A segment that decodes to something containing a
   separator is refused, not re-split.
4. **Containment is proved lexically, not against the filesystem.**
   The resolver cannot tell whether `root/uploads/x` is a symbolic link
   to `/etc/shadow`, and neither can `staticfs`: the standard library
   answers no symbolic-link question at all, with no `realpath`, no
   `readlink` and no link flag on any metadata call.
   `staticfs.containment_is_lexical_only()` answers `true` today, so a
   deployment can assert on it. A root written by a build is safe. A
   root that accepts uploads and follows links is not.
5. **Listings and dotfiles are off by default.** A listing publishes
   every name in a directory, which is how a backup file nobody meant
   to deploy is found. `.git`, `.env` and `.htpasswd` are among the
   most-scanned paths on the internet. `staticpath.with_listings` and
   `with_hidden` turn each on deliberately.
6. **A directory requested without a trailing slash gets a 301.** It is
   not cosmetic: every relative link on a page served from `/docs`
   resolves against `/`, and the same page served from `/docs/`
   resolves against `/docs/`. Skipping the redirect serves a page whose
   stylesheet and images 404.
7. **`If-None-Match` overrides `If-Modified-Since`.** RFC 9110 section
   13.2.2 says so and implementations skip it. A browser revalidating
   sends both, and a server that checks the date independently answers
   200 with a whole file to a request that deserved a 304, on every
   revalidation. `staticmeta.evaluate` is the whole precedence table in
   one function.
8. **`Vary: Accept-Encoding` goes on every response**, including the
   uncompressed ones, because the header describes the URL rather than
   the response that happened to be chosen. A shared cache that stored
   a gzip response under the URL alone serves those bytes to the next
   client, including one that cannot decode them, and a stylesheet then
   renders as binary for everybody behind that cache until it expires.
   `staticenc.vary_header()` is the value.
9. **Bound a multi-range request.** `bytes=0-0,1-1,2-2,...` with ten
   thousand ranges produces a multipart response many times the size of
   the file, and repeating one range is a plain multiplier.
   `StaticRangeLimits` is a ceiling on both, checked at parse.
10. **A failed `If-Range` means serve the whole file with a 200, not a
    412** (RFC 9110 section 13.1.5). That is what makes a resumed
    download safe. A weak validator never satisfies it, because
    equivalent for caching is not the same bytes, and splicing a range
    from a different representation into a client's buffer produces a
    file that is corrupt in the middle with a valid length.
11. **The validator is a content hash.** The standard library answers
    no modification time, so the usual size-and-modification-time weak
    validator cannot be computed here at all. `staticmeta.entry`
    computes SHA-256 over the bytes, which is a strong validator and
    therefore usable for `If-Range`. That cost is why `staticfs.scan`
    exists: once per file at start-up, then a comparison per request.
    `staticmeta.entry_with_etag` takes a validator the build already
    computed.
12. **`Last-Modified` is emitted only from a time the caller
    supplied.** With no modification time from the filesystem there is
    nothing truthful to put in it, and it is absent otherwise. `ETag`
    and `If-None-Match` do the whole job, and section 13.1.3 makes
    `If-Modified-Since` the weaker of the two.
13. **This package does not compress.** It serves the `.gz` or `.br`
    sibling a build already produced. Compressing per request
    compresses the same file once per requester rather than once per
    deployment.
14. **Use `staticenc.worth_compressing` in the build.** It is the same
    rule the server applies, so a deployment does not ship ten thousand
    useless `.gz` files beside its PNGs.
15. **Pick the cache preset per kind of file.** A fingerprinted asset
    and an HTML page want different answers, and using one for both is
    the mistake the presets exist to prevent.
    `staticmeta.looks_fingerprinted` answers which a path looks like.

## The presets and the defaults

| `staticmeta` preset | For | What it means |
| --- | --- | --- |
| `fingerprinted()` | A file whose name carries a content hash | cached for a long time and never revalidated |
| `revalidated()` | A file whose name is stable and whose content changes | stored, and revalidated before use; with an `ETag` that is a 304 with no body |
| `uncached()` | Something a shared cache must not keep | private, and not stored |
| `cached_for(n)` | A caller's own age | `max-age=n` |

| `StaticPolicy` field | `defaults()` |
| --- | --- |
| `index_names` | `index.html` |
| `list_directories` | false |
| `serve_hidden` | false |
| `redirect_directories` | true |
| `max_segments` | 32 |
| `max_bytes` | 1024 |

| `StaticRangeLimits` field | `default_limits()` |
| --- | --- |
| `max_ranges` | 4 |
| `max_total_percent` | 150, meaning one and a half times the file |

A video player sends one range and a parallel downloader a handful. The
second bound is a multiple rather than an absolute number because what
is being bounded is amplification.

## What is not included

- **Compression.** See rule 13. flate-nv and brotli-nv are what a build
  uses, and this package does not depend on either.
- **Following symbolic links, or proving one is not followed.** See
  rule 4.
- **A modification time.** See rule 11.
- **Writing to a socket.** This package answers a value and the
  caller's server writes it.
- **Uploads, or anything that writes to the root.**
- **A build for a microcontroller.** Something has to read the disk.

## Related packages

- [mime-nv](https://novo-lang.org/packages/mime-nv) answers the
  `Content-Type` for a file, and the sniffing rules a static file
  server is the one program that can sensibly set `nosniff` for. This
  package depends on it.
- [router-nv](https://novo-lang.org/packages/router-nv) matches the
  request path that decides whether a request reaches this package at
  all, and makes the same argument about not decoding before matching.
- [session-nv](https://novo-lang.org/packages/session-nv) and
  [oauth2-nv](https://novo-lang.org/packages/oauth2-nv) are the rest of
  a web tier's host half.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the
  civil date an HTTP date is formatted from and parsed into. This
  package depends on it.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies the
  SHA-256 under the validator. This package depends on it.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) reads
  the request line, the header fields and the body's framing. It does
  not parse `Range`, `If-None-Match`, `If-Modified-Since` or
  `Accept-Encoding`, which is why those four are parsed here.
- `std.http_server` in the standard library resolves a path with
  `url_to_fs_path`, whose containment check is a search for `..`. That
  check passes `%2e%2e`, `%2F`, a NUL byte and a backslash, and refuses
  `a..b.txt`, which is a real file name. Around it there is no entity
  tag, no conditional request, no range, no `Accept-Encoding` and no
  `Cache-Control`, and reading a file reads all of it to answer any
  part.

## Tests

```bash
novo test tests/staticpath_tests.nv    # every refusal, with the input that triggers it
novo test tests/staticmeta_tests.nv    # validators, HTTP dates, the precedence table
novo test tests/staticrange_tests.nv   # the Range grammar and the limits
novo test tests/staticenc_tests.nv     # Accept-Encoding and sibling choice
novo test tests/staticfs_tests.nv      # the root, the index and the serve calls
```

The normative sources are RFC 9110 section 8.8.3 for entity tags,
section 13 for conditional requests, section 14.2 for ranges and
section 12.5.3 for `Accept-Encoding`, and RFC 9111 section 5.2 for
`Cache-Control`. The reference implementations are tower-http's
`ServeDir` and WhiteNoise.

`tests/staticpath_tests.nv` is the resolver's rule set as a readable
file: every refusal with the input that triggers it, against a
predicate backed by a list of three names rather than by a disk.
Nothing in it performs. It asserts that percent-encoded dot segments
are refused, that a segment decoding to something with a separator is
refused rather than re-split, that a dotfile and a directory listing
are refused under the default policy, and that a directory without a
trailing slash answers a redirect.

The other suites assert that `If-None-Match` suppresses the
`If-Modified-Since` check, that a weak validator does not satisfy
`If-Range`, that a range list past either limit is refused at parse,
and that `Vary` is emitted on an uncompressed response.

The tests compile today and fail at run, each on the
`not implemented: static-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `pub struct` and `pub enum` in the six modules | the types are declared |
| `staticerr.refusal_status`, `.status_for`, `.refusal_detail`, `.looks_like_probing` | no |
| `staticerr.is_configuration_fault`, `.refusal_names`, `StaticFault.message` | no |
| `staticpath.defaults`, `.with_listings`, `.with_index_names`, `.with_hidden` | no |
| `staticpath.resolve`, `.check`, `.relative_of`, `.is_servable`, `.join_root` | no |
| `staticpath.kind_missing`, `.kind_file`, `.kind_directory` | no |
| `staticpath.segments`, `.check_segment`, `.has_separator`, `.has_forbidden_byte` | no |
| `staticpath.is_dot_segment`, `.is_hidden`, `.is_well_known`, `.directory_redirect` | no |
| `staticmeta.entry`, `.entry_with_etag`, `.with_modified`, `.has_modified`, `.with_type` | no |
| `staticmeta.quote_etag`, `.quote_etag_weak`, `.etag_strong_eq`, `.etag_weak_eq`, `.parse_etag_list` | no |
| `staticmeta.format_http_date`, `.parse_http_date`, `.http_date_formats` | no |
| `staticmeta.no_conditions`, `.conditions_of`, `.evaluate`, `.range_allowed` | no |
| `staticmeta.fingerprinted`, `.revalidated`, `.uncached`, `.cached_for`, `.cache_header`, `.looks_fingerprinted` | no |
| `staticrange.default_limits`, `.limits`, `.parse`, `.is_byte_range` | no |
| `staticrange.total_bytes`, `.range_len`, `.content_range`, `.unsatisfiable_header`, `.accept_ranges` | no |
| `staticrange.multipart_type`, `.part_header`, `.multipart_end`, `.multipart_len` | no |
| `staticrange.resolve_closed`, `.resolve_open`, `.resolve_suffix`, `.any_satisfiable` | no |
| `staticenc.none_available`, `.available`, `.with_preference`, `.choose` | no |
| `staticenc.quality_of`, `.identity_refused`, `.is_refused` | no |
| `staticenc.coding_name`, `.coding_suffix`, `.sibling_path`, `.coding_of_path` | no |
| `staticenc.vary_header`, `.not_acceptable_status`, `.worth_compressing`, `.producing_packages` | no |
| `staticfs.root`, `.with_policy`, `.with_ranges`, `.without_encoding` | no |
| `staticfs.scan`, `.scan_at`, `.index_len`, `.lookup`, `.encodings_of`, `.index_paths` | no |
| `staticfs.kind_of`, `.stat_one`, `.siblings_of`, `.read_range`, `.read_all`, `.list_dir` | no |
| `staticfs.serve`, `.serve_live`, `.served_header`, `.containment_is_lexical_only` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
