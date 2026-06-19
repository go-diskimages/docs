# Components

`go-diskimages` is a set of dependency-light Go modules (standard library only,
`CGO_ENABLED=0`) arranged around one small disk-image **format contract**.

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`interface`](interface.md) | `github.com/go-diskimages/interface` | contract | The `Format` contract (`Name`/`Create`/`Detect`/`ToRaw`/`Resize`, composed from `Creator`/`Detector`/`RawConverter`/`Resizer`) that every codec satisfies **structurally** — codecs need not import it. |
| [`diskimage`](diskimage.md) | `github.com/go-diskimages/diskimage` | toolkit | Unified toolkit + `cmd/diskimage` CLI: create raw, QCOW2→raw, OCI/Tart disk extraction, GRUB patching, ext4 label edit, and one `OpenBlockDevice` dispatcher (with optional LUKS) over raw / QCOW2 / UDIF-DMG. |
| [`qcow2`](qcow2.md) | `github.com/go-diskimages/qcow2` | codec | QCOW2 v2/v3 codec: create, detect, convert-to-raw (deflate clusters), grow-resize, and a lazy copy-on-write `Device`. Big-endian on-disk. |
| [`dmg`](dmg.md) | `github.com/go-diskimages/dmg` | codec | Apple UDIF/DMG helpers: detect variant, convert between UDRW/UDRO/UDSP/UDZO, resize, wrap/unwrap raw. Big-endian `koly` trailer + `blkx` mish tables. |

Each codec satisfies the [`interface`](interface.md) `Format` contract so the
[`diskimage`](diskimage.md) toolkit can treat them polymorphically.

The org additionally reserves two **empty** placeholder repos — `raw` and
`tart-oci` — for a standalone raw codec and an OCI/Tart extractor. They ship no
code yet; both code paths currently live inside [`diskimage`](diskimage.md).
