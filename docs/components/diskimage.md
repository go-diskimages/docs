# diskimage — toolkit + CLI

`github.com/go-diskimages/diskimage` is the unified toolkit for **creating and
converting** VM disk images (raw / QCOW2 / OCI-Tart) and **patching GRUB**
configurations in place. It is used to prepare images for Apple
Virtualization.framework VMs. It also ships a `cmd/diskimage` CLI (renamed from
the older `diskimagec`).

```
github.com/go-diskimages/diskimage
```

It builds on the [`qcow2`](qcow2.md) and [`dmg`](dmg.md) codecs and on the
sibling `ext4` (go-filesystems) and `lzfse` (go-compressions) packages. OCI and
GRUB features are macOS-only (`darwin` build tag).

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

## Image creation & conversion

```go
// Raw images.
func CreateRaw(path string, sizeBytes int64) error

// QCOW2 → raw.
func IsQCOW2File(path string) bool
func ConvertQCOW2ToRaw(src, dst string, w io.Writer) error
```

## OCI / Tart disk extraction *(darwin)*

```go
const TartDiskMediaType = "application/vnd.cirruslabs.tart.disk.v2"

func BlobPath(cacheDir, digest string) string
func ExtractOCIDisk(cacheDir, dst string, w io.Writer) error
```

## GRUB patching *(darwin)*

```go
func PatchGrubQuiet(diskPath string)    // removes `quiet` from the GRUB cmdline
func PatchGrubConsole(diskPath string)  // enables VGA + VirtIO console
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

macOS only for the OCI and GRUB features (`darwin` build tag); the raw / QCOW2 /
DMG block-device and conversion paths build everywhere.

## Debugging

| Env var | Effect |
|---------|--------|
| `DISKIMAGE_DEBUG` | global debug output |
| `DISKIMAGE_ZFS_DEBUG` | ZFS-specific traces |
| `DISKIMAGE_BTRFS_DEBUG` | Btrfs-specific traces |
