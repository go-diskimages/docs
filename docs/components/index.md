# Components

`go-diskimages` is a set of dependency-light Go modules (standard library only,
`CGO_ENABLED=0`) arranged around one small disk-image **format contract**.

| Module | Import path | Layer | What it does |
|--------|-------------|-------|--------------|
| [`interface`](interface.md) | `github.com/go-diskimages/interface` | contract | The `Format` contract (`Name`/`Create`/`Detect`/`ToRaw`/`Resize`, composed from `Creator`/`Detector`/`RawConverter`/`Resizer`) that every codec satisfies **structurally** — codecs need not import it. |
| [`diskimage`](diskimage.md) | `github.com/go-diskimages/diskimage` | toolkit | Unified toolkit + `cmd/diskimage` CLI: create/convert/resize images, in-image file operations, filesystem detection, ext4 label edit, and block devices (with optional LUKS/APFS-FDE decryption) over raw / QCOW2 / UDIF-DMG. |
| [`qcow2`](qcow2.md) | `github.com/go-diskimages/qcow2` | codec | QCOW2 v2/v3 codec: create, detect, convert-to-raw (deflate clusters), grow-resize, and a lazy copy-on-write `Device`. Big-endian on-disk. |
| [`dmg`](dmg.md) | `github.com/go-diskimages/dmg` | codec | Apple UDIF/DMG helpers: detect variant, convert between UDRW/UDRO/UDSP/UDZO, resize, wrap/unwrap raw. Big-endian `koly` trailer + `blkx` mish tables. Pure Go, cross-platform (no `hdiutil`). |
| [`raw`](raw.md) | `github.com/go-diskimages/raw` | codec | The unstructured identity format: create, detect (negative/fallback), identity convert, grow/shrink, and a read-write `Device`. |
| [`tart-oci`](tart-oci.md) | `github.com/go-diskimages/tart-oci` | codec | Pulls [Tart](https://github.com/cirruslabs/tart) VM images from an OCI registry, verifies every blob by digest, and materializes a byte-exact raw disk. |

Each codec satisfies the [`interface`](interface.md) `Format` contract so the
[`diskimage`](diskimage.md) toolkit can treat them polymorphically. GRUB
config patching lives in the sibling `go-bootloaders/grub` package, not in
this org.
