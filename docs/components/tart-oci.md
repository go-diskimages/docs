# tart-oci — Tart/OCI disk puller

`github.com/go-diskimages/tart-oci` (Go package `tartoci`) pulls
[Tart](https://github.com/cirruslabs/tart) VM images stored as OCI artifacts
in a registry (e.g. `ghcr.io/cirruslabs/*`). It speaks the OCI distribution
(registry v2) API, verifies every blob against its digest, decompresses the
disk layers, and materializes a byte-exact raw disk — pure Go (`CGO_ENABLED=0`),
no `tart`, no `hdiutil`, no darwin-only code.

## Tart's OCI layout

A Tart image is an OCI image manifest whose layers are:

| Media type | Contents |
|---|---|
| `application/vnd.cirruslabs.tart.config.v1` | VM config (JSON) |
| `application/vnd.cirruslabs.tart.disk.v2` | one disk chunk of ≤ 512 MiB (uncompressed), **Apple LZ4 frame** |
| `application/vnd.cirruslabs.tart.nvram.v1` | NVRAM store (raw bytes) |

Each disk layer carries `org.cirruslabs.tart.uncompressed-size` and
`org.cirruslabs.tart.uncompressed-content-digest` annotations; the chunks
concatenate, in manifest order, into the full disk. Compression is Apple's
Compression framework LZ4 **frame** format (`bv41`/`bv4-`/`bv4$` blocks
sharing a 64 KiB cross-block window) — decoded via
[`go-compressions/lz4`](https://github.com/go-compressions/lz4)
(`DecompressAppleStream`).

## Package functions

```go
// Pull a Tart image and write its raw disk image.
err := tartoci.PullDisk(ctx,
    "ghcr.io/cirruslabs/macos-sequoia-base:latest",
    "disk.raw", os.Stdout)

// Or materialize a full VM bundle (disk.img + config.json + nvram.bin).
err := tartoci.Pull(ctx,
    "ghcr.io/cirruslabs/macos-sequoia-base:latest",
    "vm/", os.Stdout)
```

Lower-level building blocks are exported too: `ParseReference`, `NewRegistry`
(`Manifest`, blob access, token auth), `BlobPath`, `ExtractDisk`, the
`Manifest`/`Descriptor` types, and the `MediaType*`/`Ann*` constants.

Options: `WithHTTPClient`, `WithBaseURL`, `WithBasicAuth` (for private repos).

### Verification

Every disk layer is checked against its `sha256` blob digest, its declared
uncompressed size, and its uncompressed content digest before its bytes are
trusted; config and NVRAM blobs are checked against their digests. A
digest/size mismatch, an auth failure, or a malformed manifest is returned as
an error — none of those paths silently succeed.

## Composition with go-diskimages

`Format` structurally satisfies the [`interface`](interface.md) `Format`
contract (`Name`/`Create`/`Detect`/`ToRaw`/`Resize`) over a locally-cached
Tart **OCI layout directory**, so a pulled image composes with the rest of
the `go-diskimages` family (`raw`, `qcow2`, `dmg`, …).
