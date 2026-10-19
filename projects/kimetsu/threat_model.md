# Kimetsu threat model

## Project and deployment

Kimetsu is a Rust workspace providing persistent memory for coding agents. It
stores project and optional user memories in SQLite, exposes a local stdio MCP
server and CLI, and provides an optional HTTP MCP server (`kimetsu-remote`) for
shared deployments. It also includes a terminal coding agent, host integration
templates, and a TypeScript client SDK.

The local CLI normally runs with the developer's OS permissions. The remote
server is a separate trust boundary: network clients must authenticate, and a
repository-scoped token must not gain access to other repositories or operations
reserved for global tokens. Operator configuration and explicitly configured
global tokens are trusted; client request fields and repository content are not.

## Untrusted input and priority surfaces

- `crates/kimetsu-remote`: HTTP headers, bearer tokens, URL repository identifiers,
  JSON-RPC/MCP requests, pagination and size parameters, and memory content.
  Prioritize authentication bypass, authorization inconsistencies, cross-repo
  access, access to shared org/user memory, path traversal, symlink escapes,
  resource exhaustion, and credential disclosure through errors or logs.
  Server-side ingest clones operator-configured repositories: their contents
  remain untrusted even though the configured clone locations are trusted.
- `crates/kimetsu-brain`: imported memory packs, sync data, repository files,
  retrieved text, provenance, and query strings. Examine parsing and size limits,
  SQL construction, scope enforcement, import/replace semantics, and handling of
  malformed or adversarial stored data. Check redaction on ingest and export;
  demonstrate a concrete secret disclosure rather than assuming arbitrary text
  can always be classified as a secret.
- `crates/kimetsu-chat`: MCP method dispatch, tool arguments, host configuration
  edits, bridge imports/exports, and integration templates in `assets/`. Examine
  whether untrusted content can cause unintended filesystem writes, command
  execution, or credential disclosure.
- `crates/kimetsu-cli` and `crates/kimetsu-core`: URL/file imports, compressed
  payloads, project discovery, state paths, configuration parsing, and subprocess
  arguments. Check decompression limits, path containment, symlinks, and handling
  of credentials embedded in URLs or configuration.
- `crates/kimetsu-agent`: file and shell tools, subprocess environment handling,
  and provider request/response parsing. Deliberately requested local shell
  execution is a product capability, not by itself an exploit; demonstrate how
  an attacker crosses an actual authorization or input-handling boundary.
- `npm/kimetsu-sdk`: endpoint construction, authentication headers, request
  serialization, and handling of untrusted server responses.

Local stdio MCP relies on the host process for access control and is not intended
to be exposed directly as an unauthenticated network service. Memory and repository
text must be treated as data. A prompt-injection finding should identify the
concrete attacker-controlled input and resulting unauthorized action or disclosure.

## Build and offline reproduction

The image contains `/src`, Rust dependencies in `/usr/local/cargo`, debug binaries
at `/src/target/debug/kimetsu` and `/src/target/debug/kimetsu-remote`, compiled test
executables, and installed TypeScript SDK dependencies. `KIMETSU_USER_BRAIN=0`
isolates tests from a real user's cross-project brain.

Run the Rust suite with:

```sh
cargo test --workspace --locked --offline --features pi,openclaw,tls -- --test-threads=1
```

For a smaller remote-server reproduction:

```sh
cargo test -p kimetsu-remote --locked --offline --features tls -- --test-threads=1
```

The remote integration tests are in `crates/kimetsu-remote/tests/`. Most Rust
modules also contain unit tests. The TypeScript suite runs with:

```sh
npm --prefix npm/kimetsu-sdk test
```

The image enables `pi`, `openclaw`, and `tls`; `graph` is enabled by the remote
crate. Optional neural `embeddings` support is not compiled and model weights
are not downloaded. Its source remains available for review, but model-dependent
reproductions require an additional online build/setup phase. External model
providers and hosted services are not needed for the baseline tests. Build-time
test warnings must be investigated rather than interpreted as passing results.

## Severity and reports

Rate findings by demonstrated impact and required privileges. Unauthenticated
remote code execution or broad unauthorized access to private brains warrants
critical severity. Cross-repository access by a restricted token, substantial
secret disclosure, or arbitrary file writes outside an authorized root generally
warrants high severity. Reproducible service-wide denial of service is generally
medium, with severity adjusted for exposure and persistence. A crash confined to
an explicitly invoked local operation is lower impact. Trusted administrators
already control configuration and global access; do not assume they are confined
to the privileges of a repository-scoped client.

Please include the affected revision, build features, deployment assumptions,
attacker privileges, a minimal offline reproducer using synthetic data, expected
and observed behavior, and a proposed regression test and patch where possible.
Benchmark scores and ordinary retrieval-quality differences are not security
issues without an additional security impact. Third-party service behavior is
outside this project's control; first-party misuse of dependencies is in scope.
