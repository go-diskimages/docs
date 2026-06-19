# qcow2 — QCOW2 v2/v3 codec

`github.com/go-diskimages/qcow2` (Go package `image_qcow2`) is a pure-Go
codec for the **QCOW2** disk-image format. Standard library only,
`CGO_ENABLED=0`, no `qemu-img`. All on-disk structures — the 72-byte header,
the L1/L2 cluster tables, the refcount table/blocks, and compressed-cluster
descriptors — are read and written **big-endian**, matching the QCOW2 wire
format. The magic bytes are `0x514649fb` (`QFI\xfb`).

## Package functions

```go
// Create writes a new empty QCOW2 v2 image (no allocated data clusters;
// reads from the equivalent raw image return zeros). cluster_bits = 16
// (64 KiB clusters). A 4-cluster file holds header, L1 table, refcount
// table, and refcount block.
func Create(path string, sizeBytes int64) error

// IsQCOW2File reports whether path begins with the QCOW2 magic bytes.
func IsQCOW2File(path string) bool

// ConvertToRaw reads a QCOW2 v2/v3 image and writes the equivalent raw
// disk image to dst, emitting "N%\n" progress lines to w.
//   Supported:   deflate-compressed clusters (compression type 0).
//   Unsupported: encryption, and ZSTD compression (v3 compression_type=1).
func ConvertToRaw(src, dst string, w io.Writer) error
```

## `Format`

`Format{}` satisfies the [`interface`](interface.md) `Format` contract
structurally (no import needed):

```go
type Format struct{}

func (Format) Name() string                                 // "qcow2"
func (Format) Create(path string, sizeBytes int64) error    // → Create
func (Format) Detect(path string) (bool, error)             // → IsQCOW2File
func (Format) ToRaw(src, dst string, w io.Writer) error     // → ConvertToRaw
func (Format) Resize(path string, newSizeBytes int64) error // grow only
```

`Resize` grows the image in place: it writes the new virtual size into the
header and extends the L1 table with zero (unallocated) entries when the new
size needs more L1 capacity. **Shrinking is rejected** — a smaller size could
silently discard allocated data.

## `Device` — lazy copy-on-write block access

`OpenDevice` opens a QCOW2 image as a read-write block device:

```go
dev, err := image_qcow2.OpenDevice("disk.qcow2")
if err != nil { log.Fatal(err) }
defer dev.Close()

buf := make([]byte, 4096)
dev.ReadAt(buf, 0)    // unallocated clusters read as zeros
dev.WriteAt(buf, 0)   // first write to a virtual cluster allocates it (COW append)
```

- Unallocated virtual clusters are allocated **lazily on first write**, appended
  to the end of the file (copy-on-write append), and the in-memory L1/L2 tables
  are updated.
- **Compressed clusters are readable**; writing *to* a compressed cluster
  returns an error (re-compression is not implemented).
- Supported versions are QCOW2 **v2 and v3**; encrypted images are rejected.
- Access is guarded by a `sync.RWMutex`.
