# cairn

A small OCI registry that keeps blobs on the filesystem and everything it needs
to *find* them in SQL.

The scope is deliberately one slice of the [OCI distribution
spec](https://github.com/opencontainers/distribution-spec/blob/main/spec.md):
blobs, manifests, tags and referrers. That slice exercises the whole shape of a
registry — content addressing, upload sessions, repository scoping, a manifest
graph that has to resolve, mutable names over immutable content, and the one query
content addressing cannot answer on its own.

Enough for `oras push`, `oras pull`, `oras repo tags`, `oras attach` and
`oras discover` to work against it unmodified.

This is experimental: a learning slice of the spec, not a registry to point
production traffic at. There are no tagged releases.

Maintained by [Daniel Bae](https://github.com/MrBeldum). See [AUTHORS.md](AUTHORS.md).

## Running it

```sh
go run ./cmd/cairnd -addr 127.0.0.1:5050 -root data
```

| Flag | Default | Meaning |
|------|---------|---------|
| `-addr` | `127.0.0.1:5050` | address to listen on |
| `-root` | `data` | directory holding `blobs/`, `uploads/` and `cairn.db` |
| `-max-blob-size` | `1073741824` | largest blob accepted, in bytes |
| `-max-manifest-size` | `4194304` | largest manifest accepted, in bytes |
| `-log-format` | `text` | `text` or `json` |
| `-log-level` | `info` | `debug`, `info`, `warn`, `error` |

The two size ceilings are separate because the constraint differs: a blob is
streamed to disk, while a manifest is held in memory to be hashed and parsed.
A bad `-log-format` or `-log-level` fails at startup rather than falling back.

## Quickstart

No TLS, so `--plain-http` is required.

```sh
echo "hello from cairn" > note.txt
oras push --plain-http 127.0.0.1:5050/acme/widgets:v1.0.0 note.txt:text/plain
oras repo tags --plain-http 127.0.0.1:5050/acme/widgets
oras pull --plain-http 127.0.0.1:5050/acme/widgets:v1.0.0
```

Tags, referrers, digest pushes and the curl forms: [docs/usage.md](docs/usage.md).

## Docs

| | |
|---|---|
| [Usage](docs/usage.md) | oras and curl against the live server |
| [Endpoints](docs/endpoints.md) | end-1 through end-13 |
| [Architecture](docs/architecture.md) | layout, why the SQL index exists, logging |
| [Decisions](docs/decisions.md) | the choices the spec leaves open |
| [Conformance](docs/conformance.md) | pinned distribution-spec suite |
| [Benchmarks](bench/README.md) | referrer cost and concurrent attach |

## Layout

```
cmd/cairnd            wiring and the HTTP server
internal/ocihttp      request parsing, status and error codes -- no storage logic
internal/registry     the rules
internal/blobstore    committed bytes, addressed by digest
internal/uploadstore  staged bytes of uploads that have no digest yet
internal/metastore    the SQL index
internal/model        what a row holds
internal/logging      logger construction
```

Three stores, different lifetimes: immutable blobs, in-flight uploads, and an
index that holds no bytes. The rest of that argument is in
[docs/architecture.md](docs/architecture.md).

## Known gaps

- end-8a returns every tag in a repository, with no ceiling. A default page size
  would fix it and is deliberately not invented here.
- end-12 is not paginated.
- No backfill from the referrers tag schema.
- Statistics can go stale within a long run. See `sqlite.Store.Optimize`.
- Nothing is ever reclaimed. Deletes remove a row and leave the bytes.
- No authentication. Blob writes are bounded only by `-max-blob-size`.
- Blob reads through `Open` are not digest-verified (would defeat `Range`).
- No request ID. Logs go to stderr and are not rotated.

## Tests

```sh
go test ./...
./scripts/conformance.sh   # 75 passed, 4 skipped
```

The tests drive the real HTTP surface over SQLite and the filesystem. Under a
syscall sandbox that blocks `sendfile`, blob reads larger than 512 bytes can
fail — that is the sandbox, not the code.

## License

[MIT](LICENSE). If you want to change the code,
[CONTRIBUTING.md](CONTRIBUTING.md) is the contract. Vulnerabilities go to
[SECURITY.md](SECURITY.md), not to a public issue.
