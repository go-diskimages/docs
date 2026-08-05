# raw — unstructured disk image codec

`github.com/go-diskimages/raw` (Go package `image_raw`) is a pure-Go
creator, detector, converter, and read-write block device for the **raw**
(unstructured) disk-image format. No CGO, no external tools. Works on all
platforms and all supported Go architectures.

## Format

A raw image is a plain byte-for-byte image of a block device: the file's
contents **are** the disk contents. There is no header, no metadata, no
allocation table, and no compression — the trivial *identity* format of the
`go-diskimages` family:

- **Create** writes a sparse, zero-filled file of the requested size.
- **ToRaw** is a straight byte copy — a raw image already *is* its own raw form.
- **Detect** is the negative/fallback case: a regular, non-empty file that
  carries none of the known structured-container signatures (QCOW2, VMDK,
  VDI, VHD, or Apple UDIF/DMG) is a raw image.
- **Resize** truncates the file up or down; both grow (zero-filled tail) and
  shrink (discarded tail) are safe, since there is no internal structure to
  corrupt.

## Package functions

```go
// Create writes a new empty raw disk image with the given size in bytes.
// The file is sparse and reads back as zeros.
func Create(path string, sizeBytes int64) error

// IsRawImage reports whether path holds a raw disk image.
func IsRawImage(path string) (bool, error)

// ConvertToRaw copies the raw image at src to dst (the identity conversion),
// emitting "N%\n" progress lines to w.
func ConvertToRaw(src, dst string, w io.Writer) error
```

## `Format`

`Format{}` satisfies the [`interface`](interface.md) `Format` contract
structurally (no import needed):

```go
type Format struct{}

func (Format) Name() string                                 // "raw"
func (Format) Create(path string, sizeBytes int64) error    // new sparse raw image
func (Format) Detect(path string) (bool, error)             // negative/fallback detection
func (Format) ToRaw(src, dst string, w io.Writer) error     // identity copy
func (Format) Resize(path string, newSizeBytes int64) error // grow or shrink
```

## `Device` — read-write block access

`OpenDevice` opens a raw image as a random-access block device. Virtual
offset N maps directly onto file offset N.

```go
dev, err := image_raw.OpenDevice("disk.raw")
if err != nil { log.Fatal(err) }
defer dev.Close()

if _, err := dev.WriteAt([]byte("hello"), 1<<20); err != nil { log.Fatal(err) }
if err := dev.Sync(); err != nil { log.Fatal(err) }

buf := make([]byte, 5)
dev.ReadAt(buf, 1<<20)
```

Unlike the sparse container formats, a raw `Device`'s virtual size can both
grow and shrink via `Truncate`.
