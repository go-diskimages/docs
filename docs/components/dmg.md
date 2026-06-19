# dmg — Apple UDIF/DMG

`github.com/go-diskimages/dmg` is a pure-Go set of helpers for the native
Apple **UDIF** disk-image format (`.dmg`). No external tools (`hdiutil`,
`plutil`) are required, and it works on **all platforms** — not just macOS.

## Format

A UDIF file ends with a 512-byte **`koly` trailer (big-endian)** that records
the image variant, the data-fork offset/length, and the offset/length of an
embedded XML plist. The plist carries an array of `blkx` (mish) tables, each a
sequence of sector runs:

| Run type   | Code         | Description                            |
|------------|--------------|----------------------------------------|
| RAW        | `0x00000001` | Uncompressed sectors stored verbatim   |
| NOCOPY     | `0x00000000` | Zero-filled sectors (no stored data)   |
| FREE       | `0x7FFFFFFE` | Unallocated, treated as zeros          |
| LZFSE      | `0x80000004` | LZFSE compressed (Apple, macOS 10.12+) |
| ZLIB       | `0x80000005` | zlib-compressed sectors                |
| Terminator | `0xFFFFFFFF` | End-of-table marker                    |

Image variants:

| Name   | Code | Read | Write | Description                            |
|--------|------|------|-------|----------------------------------------|
| `UDRW` | 1    | ✓    | ✓     | Read-write, fixed size                 |
| `UDRO` | 2    | ✓    | ✓     | Read-only (stored as RAW)              |
| `UDCO` | 3    | ✓    | —     | ADC compressed (write not supported)   |
| `UDZO` | 4    | ✓    | ✓     | zlib compressed (1 MiB chunks)         |
| `UDBZ` | 5    | ✓    | —     | bzip2 compressed (write not supported) |
| `UDSP` | 11   | ✓    | ✓     | Sparse (NOCOPY runs for zero sectors)  |

## Public API

```go
// DetectUDIFFormat returns "UDRW", "UDSP", … from the koly trailer.
func DetectUDIFFormat(path string) (string, error)

// IsUDIF reports whether path is a valid Apple UDIF image.
func IsUDIF(path string) bool

// ConvertUDIF reads all sectors from src and writes them to dst in dstFormat.
// Write formats: UDRW, UDRO, UDSP (sparse), UDZO (zlib). UDCO and UDBZ are
// not yet writable. CRC-32 checksums are written in the koly + blkx headers.
// Multi-segment source images are not supported.
func ConvertUDIF(src, dst, dstFormat string) error

// ResizeUDRW grows a UDIF image to newSizeBytes (rounded to a 512-byte
// boundary), preserving the variant. Shrinking is not supported.
func ResizeUDRW(path string, newSizeBytes int64) error

// UnpackToTemp extracts all sectors to a raw temp file (caller os.Removes it).
func UnpackToTemp(path string) (tmpPath string, err error)

// PackFromTemp wraps the raw file as a new UDIF UDRW image (atomic).
func PackFromTemp(tmpPath, destPath string) error

// WrapRaw converts an existing raw file into a UDIF UDRW image in place (atomic).
func WrapRaw(path string) error
```

## `Format`

`Format{}` satisfies the [`interface`](interface.md) `Format` contract
structurally (no import needed):

```go
type Format struct{}

func (Format) Name() string                                 // "dmg"
func (Format) Create(path string, sizeBytes int64) error    // blank UDIF UDRW
func (Format) Detect(path string) (bool, error)             // koly magic
func (Format) ToRaw(src, dst string, w io.Writer) error     // extract sectors → raw
func (Format) Resize(path string, newSizeBytes int64) error // grow or shrink
```

## Examples

```go
import "github.com/go-diskimages/dmg"

// Detect and convert.
variant, _ := dmg.DetectUDIFFormat("disk.dmg")        // e.g. "UDRW"
_ = dmg.ConvertUDIF("disk.dmg", "disk.sparse.dmg", "UDSP")
_ = dmg.ConvertUDIF("disk.dmg", "disk.z.dmg", "UDZO")
_ = dmg.ResizeUDRW("disk.dmg", 2<<30)                 // grow to 2 GiB

// Wrap a raw file, edit, repack.
_ = dmg.WrapRaw("disk.raw")                           // disk.raw → UDIF in place
tmp, _ := dmg.UnpackToTemp("disk.dmg")
defer os.Remove(tmp)
_ = dmg.PackFromTemp(tmp, "disk.dmg")
```
