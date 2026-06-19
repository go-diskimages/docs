# go-diskimages

Pure-Go **disk-image tooling** — create, detect, convert, and resize VM disk
images with **no cgo and no external tools** (`hdiutil`, `qemu-img`, `plutil`
are not required).

Each supported on-disk format (raw, **QCOW2**, Apple **UDIF/DMG**) is a small,
dependency-light Go module that speaks one shared **format contract**: `Name`,
`Create`, `Detect`, `ToRaw`, and `Resize`. The `diskimage` toolkit unifies them
behind a single block-device dispatcher and a `cmd/diskimage` CLI, so callers
manipulate an image by path without caring which container it is in.

The codecs are deliberately faithful to the wire format. QCOW2 is decoded
straight from its **big-endian** on-disk structures (header, L1/L2 tables,
refcount tables, deflate-compressed clusters); DMG is parsed from its
**big-endian `koly` trailer** and embedded `blkx`/mish plist tables. No data
is round-tripped through a third-party tool.

It pairs with the sibling storage orgs — [go-volumes](https://github.com/go-volumes)
(block backings) and [go-filesystems](https://github.com/go-filesystems) (on-disk
filesystem formats) — and feeds image preparation for Apple
Virtualization.framework VMs.

## Components

| Module | Layer | What it does |
|--------|-------|--------------|
| [`interface`](components/interface.md) | contract | The shared format contract (`Name`/`Create`/`Detect`/`ToRaw`/`Resize`) every codec satisfies structurally — no import required. |
| [`diskimage`](components/diskimage.md) | toolkit | Unified create/convert toolkit + `cmd/diskimage` CLI; one `OpenBlockDevice` dispatcher over raw / QCOW2 / UDIF-DMG, optional LUKS, ext4 label edit, GRUB patching. |
| [`qcow2`](components/qcow2.md) | codec | QCOW2 v2/v3 codec — create, detect, convert-to-raw, resize, and a lazy copy-on-write block device. Big-endian on-disk. |
| [`dmg`](components/dmg.md) | codec | Apple UDIF/DMG helpers — detect, convert between variants, resize, wrap/unwrap raw. Big-endian `koly` trailer. |

!!! note "Placeholders not yet shipped"
    The org also holds two **empty placeholder** repos — `raw` and `tart-oci` —
    reserved for a standalone raw-image codec and an OCI/Tart disk extractor.
    They contain no code yet and are intentionally omitted here. (Today the
    raw and Tart-OCI paths live inside [`diskimage`](components/diskimage.md).)
