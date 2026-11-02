# Changelog

All notable changes to static-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-11

The implementation.  Every `todo()` body in the six modules is
replaced, `stability` is `"experimental"`, and the manifest floor is
`novo = ">= 0.20.2"`, the toolchain every measurement below was taken
on.  `novo test --isolate tests` is green — 105 tests in six modules,
and no leak line — and merged line coverage of `src/` is 999 of 1002
lines, the three left excused in the source.

### Added

- `tests/static_edge_tests.nv`, 26 tests — the sixth suite, and the only
  one that writes anything.  Its first half builds a root under
  `fs.temp_dir()` holding a file, a precompressed sibling, a dotfile, a
  symbolic link and a subdirectory, and drives a scan, a lookup, a range
  read, a listing and the two serve calls through it.  Its second half
  reaches the arms a table-driven suite leaves: the detail, the status
  and the probing verdict of all nine refusals, the codings and suffixed
  paths `staticenc` answers, the conditional forms the precedence table
  does not name, the range grammar the parser refuses, and an index
  built by hand so a weak validator reaches the multipart path without a
  file to compute one from.

### Changed

- **The 0.0.x statements about what the standard library cannot answer
  are corrected here rather than rewritten.**  `std.fs` answers a file's
  modification time and it answers symbolic-link questions — `realpath`,
  `is_symlink` and `read_link` — as of novo 0.10.0, so `staticfs` dates
  every scanned entry from `fs.file_mtime` and a listing carries a real
  `Last-Modified`, and the missing containment check is now a decision
  this package makes rather than one the platform forced.  The 0.0.1
  design note about replacing "the standard library's thirteen-line
  handler" named a thing the standard library does not have:
  `std.http_server` routes and writes a response, and serves no file.

### Known

- **Containment is lexical only, and it is measured.**
  `staticfs.containment_is_lexical_only()` answers `true`, and
  `tests/static_edge_tests.nv` puts a symbolic link inside the root
  pointing at a file outside it: every segment is one the resolver
  allows, the path is inside the root as a string, and the bytes that
  come back are not.  A root a deployment controls end to end is safe;
  a root a stranger can write into is the one where this costs
  something, and a deployment in that position asks `fs.is_symlink` on
  each path or holds its files where a link cannot be made.
- **Three arms are excused where they stand.**  No filesystem this
  package can reach produces them: `staticfs.clock_time`'s refusal of
  three integers that are in range by construction, and the absent
  modification time in `dated` and `directory_stamp`, for a file and a
  directory a walk has just read.  Each carries a `cov: skip` marker
  naming the reason, and the three are named in
  `tests/static_edge_tests.nv`; `novo pkg publish --dry-run` reports
  the excused regions itself.
- **`fs.read_bytes` on a directory aborts the process** with a huge
  allocation instead of answering `None`, and a directory named after a
  file is something a stranger who can write into a root can create;
  filed against the `novo` package as
  `stdlib/fs-read-bytes-on-a-directory-ooms-the-process` (open,
  0.20.2).  `staticfs.read_file_bytes` asks `fs.is_file` first, which is
  the check a caller reading a path it did not create should make
  regardless.  The guard stays when that is fixed.
- **No local is named after a stdlib module.**  A binding named `path`
  or `time` shadows the module — W0002 — and a `match` over it with a
  call through the module's name in an arm is an internal compiler
  error the per-module check passes; filed as
  `typecheck/a-local-that-shadows-a-stdlib-module-ices-a-match-over-it`
  (open, 0.20.2).  `on_disk` and `clock_value` are the longer names
  that avoid it, and they are the clearer names either way.

## 0.0.2 — 2026-09-18

`StaticRefusal` implements `Error`; no signature changed.  README rewritten to the package README style guide (docs/writing-a-readme.md).

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `staticpath` — the resolver, `[]` throughout, and the load-bearing
  half of the package.  `StaticResolved` has five variants because a
  static file server genuinely has five answers; `StaticPolicy` carries
  the index names, the listing and dotfile decisions and the two
  ceilings; and the nine rules are published one by one —
  `has_separator`, `has_forbidden_byte`, `is_dot_segment`,
  `is_hidden`, `check_segment` — so a reviewer can read them rather
  than trust them.
- `staticmeta` — validators and conditional requests, `[]`.  `evaluate`
  is RFC 9110 § 13.2.2's precedence table whole, including the rule
  that `If-None-Match` overrides `If-Modified-Since`; `etag_strong_eq`
  and `etag_weak_eq` are two comparisons because `If-Range` needs the
  first and `If-None-Match` the second; `parse_http_date` reads all
  three formats § 5.6.7 requires a recipient to accept, and
  `format_http_date` writes the one a sender may use.  Three
  cache-control presets, and `looks_fingerprinted` as an admitted
  heuristic.
- `staticrange` — `Range`, `[]`.  The three forms resolved separately
  so each one's arithmetic is a test: a suffix is the LAST n bytes,
  `-0` is unsatisfiable, a `last-byte-pos` past the end is the end, and
  a zero-length file answers nothing.  `StaticRangeLimits` bounds both
  the range count and the amplification; `multipart_len` exists because
  a multipart body is longer than its parts.
- `staticenc` — precompressed siblings, `[]`.  The three rules
  § 12.5.3 carries that implementations skip: `q=0` is a refusal,
  `identity` is acceptable unless refused, and `*;q=0` is the header
  that means 406.  `vary_header` and `worth_compressing` are published
  so a cache is not poisoned and a build does not gzip its PNGs.
- `staticfs` — the half that reads, `[fs]` and nothing else.  `scan`
  builds an index once at start-up; `serve` answers a whole request
  from it; `read_range` is positional, so serving the last megabyte of
  a gigabyte file reads a megabyte.  `containment_is_lexical_only`
  answers `true` and says why.
- `staticerr` — `StaticRefusal` with one variant per rule and
  `StaticFault` for the things that are actually failures.  Every
  refusal is a 404 and `refusal_detail` is the half a client never
  sees; `looks_like_probing` is the number to graph.
- `tests/` — 78 API tests against the signatures, red until the bodies
  land.  `staticpath_tests.nv` is the security argument: the whole rule
  set as a table, with no filesystem in the room.

### Known

- Every body is `todo()`.  `novo test` is red, `novo pkg build` is
  green, and the shard rows that measure the design —
  `effect-budget`, `dep-layer`, `no-discharge-in-core` — pass.
- **Containment is lexical only.**  The standard library answers no
  symbolic-link question, so neither this package nor anything built on
  it can prove a resolved path stays inside its root once a symbolic
  link is involved.  `staticfs.containment_is_lexical_only` says so.
- **No `Last-Modified` unless the caller supplies a time.**  The
  standard library answers no file modification time either, so there
  is nothing truthful to put in the header by default.
- No `tests/embedded_probe.nv`: this is a `host` package, so it makes
  no device claim to check.

### Design notes

`Range`, `If-None-Match`, `If-Modified-Since` and `Accept-Encoding` are
parsed here because nothing else on the registry parses them.
http-codec-nv reads a request line, the header fields as name-value
pairs and the body's framing, and stops. mime-nv parses `Accept` and
cannot parse `Accept-Encoding`: RFC 9110 section 12.5.3's grammar is a
list of codings with quality values where section 12.5.1's is a list of
media ranges with them, and `gzip` has no `/` in it. A second consumer,
such as a caching proxy or a client that revalidates its own cache,
would be the reason to move those four into a shared package.

Adopting this in `orbit/website` would replace the standard library's
thirteen-line handler: `staticfs.scan` at start-up and `staticfs.serve`
per request would give the documentation pages 304s instead of full
re-downloads, serve the `.gz` files the generator can already write,
and read a byte range of the wasm blob rather than all of it. The
choice to make first is the cache policy: fingerprinted assets want
`staticmeta.fingerprinted` and HTML pages want
`staticmeta.revalidated`.
