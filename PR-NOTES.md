# PR notes — KV: reject empty subject tokens in keys, validate search keys

Port of nats.go #2076 to `async-nats`, plus the search-key validator Go already had.

## Branch / commit

    git checkout -b fix/kv-consecutive-dots
    git add async-nats/src/jetstream/kv/mod.rs async-nats/tests/kv_tests.rs
    git commit

Suggested commit message (tbaggery style, as CONTRIBUTING requires):

    Reject KV keys with empty subject tokens

    KV keys become NATS subject tokens. is_valid_key already rejected an
    empty key and a leading or trailing `.`, but then handed off to a regex
    where `.` is an ordinary permitted character, so `a..b` passed. The
    resulting subject has an empty token, which the server refuses, so the
    caller saw a late and misleading failure: put() reported
    `Ack: no stream found for given subject` and create() reported
    `503 no responders`, both of which read as "the bucket is missing".

    Reject `..` at the client boundary, matching nats.go #2076.

    Watch paths take filters rather than plain keys, so they need their own
    rule: the same empty-token check plus the NATS wildcards `*` and a
    trailing `>`. Add is_valid_search_key and apply it on both watch
    chokepoints, which finally constructs the WatchErrorKind::InvalidKey
    variant that until now was declared but never produced.

## Scope

`async-nats/src/jetstream/kv/mod.rs`
- `has_non_empty_tokens` — shared empty-token rule (empty, leading `.`, trailing `.`, `..`).
- `is_valid_key` = empty-token rule + `VALID_KEY_RE` (unchanged regex).
- `is_valid_search_key` = empty-token rule + new `VALID_SEARCH_KEY_RE`
  `\A[-/_=\.a-zA-Z0-9*]*[>]?\z`, byte-for-byte the Go `validSearchKeyRe`.
- `watch_with_deliver_policy` and `watch_many_with_deliver_policy` validate with
  `is_valid_search_key`. Every public watch entry point funnels through one of these two
  (`watch`, `watch_with_history`, `watch_from_revision`, `watch_all`,
  `watch_all_from_revision`, `watch_many`, `watch_many_with_history`), so this is the whole
  surface. `watch_all*` pass `>`, which the search regex accepts.
- The five `InvalidKey` Display strings now say `key cannot be empty, start or end with
  `.`, or contain `..``. The existing `create_with_failure` test asserts on the
  `key cannot be empty` prefix, which is preserved.

`async-nats/tests/kv_tests.rs`
- `invalid_key_rejected_at_client` — bad keys rejected on put/create/update/entry/
  delete/purge/history; `a.b`, `foo.bar.baz`, `a-b_c=d/e`, `plain` still round-trip.
- `invalid_search_key_rejected_at_client` — bad filters rejected on all five watch entry
  points; `>`, `*`, `foo.>`, `foo.*`, `*.bar`, `foo.*.bar`, `a.b.c` still accepted.

## Breaking-change note for the PR body

This turns previously-late server errors into an immediate client-side
`ErrorKind::InvalidKey`. Any caller that today reaches the server with `a..b` is already
broken; the only user-visible change is *which* error they get and how early. Worth calling
out in the PR body so it lands in the right release notes section.

## Deliberately out of scope (mention in the PR, do not fix here)

`Store::history` and `Store::get`/`entry` use `is_valid_key`, so they reject wildcards. Go's
`History` routes through `WatchFiltered` and therefore uses `searchKeyValid`, i.e. Go accepts
`history("foo.*")` where Rust does not. That is a pre-existing divergence, unrelated to the
empty-token bug, and loosening it is a behaviour change that deserves its own PR.

## CI checklist (all run locally, all green)

    cargo +nightly fmt --all -- --check
    cargo clippy -p async-nats --benches --tests --examples --all-features -- --deny clippy::all
    cargo test -p async-nats --lib --all-features          # 90 passed, incl. 4 new validator tests
    cargo test -p async-nats --test kv_tests --all-features # 33 passed, 1 ignored
    cargo doc -p async-nats --no-deps --all-features

Local gotcha, not something to change in the PR: the `jsonschema` dev-dependency pulls
`reqwest -> native-tls -> openssl-sys`, which needs libssl headers. Without `libssl-dev`,
temporarily add `openssl = { version = "0.10", features = ["vendored"] }` to
`async-nats/[dev-dependencies]` to get the test targets to build, and drop it before committing.

## Negative control

The two new integration tests were run against unpatched `kv/mod.rs`
(`git stash push -- async-nats/src/jetstream/kv/mod.rs`) and both fail:

    kv::invalid_key_rejected_at_client ... FAILED   assertion failed: put("a..b")
    kv::invalid_search_key_rejected_at_client ... FAILED

## Upstream reference

- nats.go PR: https://github.com/nats-io/nats.go/pull/2076
- Link it in the PR body; nats.rs asks that changes be discussed in an issue first, so open
  one referencing the Go PR if none exists.