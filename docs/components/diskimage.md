# diskimage — toolkit + CLI

`github.com/go-diskimages/diskimage` is the unified toolkit (library + CLI) for
**creating, converting, resizing**, and reading/writing files inside VM disk
images (raw / DMG-UDIF, plus QCOW2 via the CLI), across
ext4/fat32/btrfs/xfs/zfs/exfat/apfs and MBR/GPT partition tables, with optional
LUKS and APFS FileVault encryption. It is used to prepare images for Apple
Virtualization.framework VMs. It also ships a `cmd/diskimage` CLI (renamed from
the older `diskimagec`).

```
github.com/go-diskimages/diskimage
```

It composes the [`qcow2`](qcow2.md) and [`dmg`](dmg.md) codecs, the
[go-filesystems](https://github.com/go-filesystems) drivers, and
[go-fde](https://github.com/go-fde) (APFS/LUKS encryption) into one CLI +
library — see its `go.mod` for exact versions. GRUB-config patching lives in
the sibling [`go-bootloaders/grub`](https://go-bootloaders.github.io/docs/)
package; OCI/Tart image extraction lives in the sibling
[`tart-oci`](tart-oci.md) package.

## Block device dispatcher

`OpenBlockDevice` opens a raw, QCOW2, or **UDIF-UDRW DMG** image behind one
uniform `BlockDevice` (`ReadAt` / `WriteAt` / `Size` / `Close`):

```go
dev, err := diskimage.OpenBlockDevice("disk.qcow2")
if err != nil { log.Fatal(err) }
defer dev.Close()

buf := make([]byte, 512)
dev.ReadAt(buf, 0)
```

Format is auto-detected: QCOW2 by its magic bytes, UDIF DMG by the `koly`
trailer, everything else as a raw file.

!!! warning "UDIF is UDRW-only and in place"
    Read/Write are bounded to the data fork (the sector area at file offset 0).
    `Close` refreshes the koly trailer's `dataForkChecksum` + `masterChecksum`
    when anything was written, so the next open passes master-checksum
    verification. Compressed subformats (UDRO / UDZO / UDBZ / UDSP) are rejected
    with a pointer to `dmg.UnpackToTemp` + `PackFromTemp` for the explicit
    unpack-edit-repack path.

### LUKS

`OpenLUKSBlockDevice` layers LUKS decryption on top of a raw or QCOW2 backing:

```go
dev, err := diskimage.OpenLUKSBlockDevice("disk.luks.qcow2", []byte("pass"))
if err != nil { log.Fatal(err) }
defer dev.Close()              // closes LUKS and the underlying device

buf := make([]byte, 4096)
dev.ReadAt(buf, 0)             // reads the decrypted LUKS payload
dev.WriteAt(buf, 0)            // encrypts and writes back
```

Backing is raw (fallback) or QCOW2 (magic `QFI\xfb`). `Size()` returns the
decrypted payload length — `0` for LUKS1 (its header does not encode the
payload length), the configured length for LUKS2.

## Image creation, growing, resizing, converting

```go
func Create(opts CreateOptions) error
func Grow(path string, sizeBytes int64) error
func ResizeImage(path string, newSizeBytes int64) error
func ConvertImageFormat(path, dstFormat string) error
```

`CreateOptions` selects the container format (`FormatRaw`/`FormatDmg`),
partition scheme (`PartNone`/`PartMBR`/`PartGPT`), filesystem (`FSNone`,
`FSExt4`, `FSFat32`, `FSBtrfs`, `FSXfs`, `FSZfs`, `FSExFAT`, `FSNTFS`,
`FSApfs`), volume label, and — for DMG images — the UDIF sub-format and an
optional FileVault passphrase. `ResizeImage`/`ConvertImageFormat` are pure Go
(via the `dmg` codec) and cross-platform. QCOW2 creation is exposed by the CLI
(`diskimage create qcow2`) directly through the sibling `qcow2` package.

## In-image file operations

```go
func ReadFile(opts FileOptions) ([]byte, error)
func WriteFile(opts FileOptions, data []byte, perm os.FileMode) error
func Stat(opts FileOptions) (filesystem.Stat, error)
func Rename(opts FileOptions, newPath string) error
func DeleteFile(opts FileOptions) error
func DeleteDir(opts FileOptions) error
func MkDir(opts FileOptions, perm os.FileMode) error
func List(opts ListOptions) ([]ListEntry, error)
func DetectFilesystem(imagePath string, partIndex int) (FilesystemType, error)
```

## ext4 volume label

```go
func SetExt4Label(imagePath string, partIndex int, label string) error
func Ext4Label(imagePath string, partIndex int) (string, error)
```

Reads or writes the ext4 `s_volume_name` of the partition at `partIndex`,
working on raw, QCOW2, and UDIF-UDRW DMG inputs via `OpenBlockDevice` and an
in-package ext4 adapter. The label is capped at 16 bytes; the metadata-csum is
refreshed with the kernel-canonical CRC-32C convention
(`crc32c(~0, sb[:0x3FC])`, no final XOR). Use it **offline** — concurrent
writers may produce a torn superblock.

## Filesystem detection

`OpenBlockDevice` plus the package's `detect.go` recognise the filesystem inside
a backing image:

| Filesystem | How detected                                            |
|------------|---------------------------------------------------------|
| APFS       | NX SuperBlock magic `"NXSB"` at offset 32 of block 0    |
| ext4       | superblock magic `0xEF53` at offset 0x438               |
| FAT32      | OEM `"MSWIN4.1"` / FAT32 BPB signature in the boot sector |
| NTFS       | OEM `"NTFSIMG1"` (test) or `"NTFS    "` (real NTFS)      |
| exFAT      | OEM `"EXFAT   "` at bytes 3–10 of the boot sector        |
| btrfs      | superblock magic `_BHRfS_M` at offset 0x10040           |
| XFS        | superblock magic `XFSB` at offset 0                     |
| ZFS        | vdev label magic across blocks 0 / 256K / end           |

## Build requirements

Pure Go (`CGO_ENABLED=0`), cross-platform, all six supported 64-bit
architectures. A handful of darwin-only integration tests additionally
cross-check the pure-Go UDIF codec against real `hdiutil` output and the
`go-bootloaders/grub` patch helpers against real GRUB behavior on macOS; they
are not part of the public API and are not required to build or use the
library on any platform.

## Debugging

The `integration` stress test's iteration count is configurable via the
`DISKIMAGE_STRESS_ITERS` environment variable.
