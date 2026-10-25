# Rust HTTP parsing review

Reviewed on 25 October 2026 against upstream commit
`4fc2f83e1c8710dba772bfd4523d0efab57737b6` in Daniel Musau's public fork.

## Scope

This focused review covers request framing in `crates/common/src/http.rs`,
its body limits and read deadlines, and the local-server entry points that
use it: live mode, the question page and component review. It is not a
complete security audit of the workspace, browser integration or installers.

## Confirmed finding and fix

The chunked-body reader accepted some incomplete or malformed uploads as
complete requests:

- EOF after the zero-sized chunk, or after a trailer field, counted as the
  blank line that should end the trailer section. Read errors were also
  converted to EOF in that loop.
- The line following chunk data could contain arbitrary bytes instead of
  exactly CRLF.
- Chunk-size and trailer lines accepted bare LF instead of CRLF.

For example, a chunked request body of `1\r\na\r\n0\r\n` previously returned
a request containing `a`, despite the missing final CRLF. Similarly,
`1\r\naBAD\r\n0\r\n\r\n` incorrectly accepted `BAD` as part of the chunk
separator. A local handler could therefore act on an upload whose framing
was not complete. No remote exploit or production incident was established.

The parser now requires CRLF on chunk-size and trailer lines, reads exactly
two delimiter bytes after each non-empty chunk, and requires the blank line
closing the trailer section. EOF and read errors return `None` instead of
dispatching an incomplete request. Existing body-cap behavior is unchanged.
Valid chunk extensions and complete trailer fields remain supported.

Four regression tests cover incomplete trailer sections, malformed data
delimiters, malformed metadata line endings and valid extensions/trailers.
The three rejection tests failed before the fix; all four passed afterward.

## Follow-up observation

Chunk-size and trailer lines still use `read_line` after the 64 KiB request
head budget is removed. The payload cap therefore does not bound these
metadata allocations. The existing request deadline bounds elapsed time,
not bytes allocated. A separate change should define metadata budgets and
test oversized extensions and trailer sections. This observation is based
on code inspection; resource-exhaustion behavior was not measured here.

## Validation

The focused command is:

```sh
cargo test -p impeccable-common chunked_framing
```

The full changed crate also passes: `cargo test -p impeccable-common`
reported **50 passed, 0 failed**, with no ignored tests.

The optimized CLI build (`cargo build --release -p impeccable`) completed
successfully. The default Bun/Node suites used that binary through
`IMPECCABLE_BIN`, with Rust toolchain 1.99.1 and Node 26.8.1.

- Core Bun tests: **471 passed** with the repository-pinned Bun 1.3.13.
  An initial run on Bun 1.4.4 failed the unchanged component-review bundle's
  byte comparison; using the pinned version resolved it without asset edits.
- Core Node tests: **194 passed, 2 failed, 1 skipped**. The failures are the
  unchanged process-group tests `escalates to SIGKILL when the child ignores
  SIGTERM` and `onExit still reaches a group a shutdown is in the middle of
  stopping`. Both also failed in isolation on this machine. These tests use
  fixed startup waits and do not exercise the Rust HTTP parser; their precise
  environmental cause was not established.
- Oracle replay: **passed with zero unreviewed differences**.
- Detector: **28 passed**.
- Live helpers/contracts: **257 passed, 2 skipped**.
- Framework fixtures: **191 passed**.
- Plugin loader E2E: **4 passed** against the installed Claude Code CLI.

The default command stops after a failed core command, so the remaining
default suites were run explicitly with `node scripts/run-tests.mjs oracle
detector live framework plugin-e2e`; that run exited successfully. No oracle
goldens or generated provider assets were changed.

The dependent-crate run (`bun run test:rust-changed`) completed with
**1,450 passed, 3 failed, 9 ignored** across its reported test targets. The
three failures were:

- `a_page_that_relays_itself_for_the_capture_is_stitched`: capture returned
  `beyond-viewport` instead of `stitched`.
- `the_pixels_read_halos_and_near_opaque_glass`: the near-opaque glass finding
  was advisory where the test expected a pixel verdict above advisory.
- `the_text_geometry_rules_measure_the_text`: the expected cramped-padding
  finding for `flag-band-next-differs` was absent.

These are unchanged browser rendering/detection tests. Their fixture servers
read requests directly rather than calling the modified HTTP parser. They
were not fixed in this scoped patch, and the run does not establish a fully
passing workspace. The entire changed common crate passed in this run too.

`git diff --check` passed. A standalone rustfmt check also found formatting
differences in existing code; broad reformatting was intentionally excluded.
