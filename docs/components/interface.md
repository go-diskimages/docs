# interface — the format contract

`github.com/go-diskimages/interface` (Go package `diskimage_format`) is the
shared **disk-image format contract** for the family. It depends only on `io`,
builds with `CGO_ENABLED=0`, and defines what it means to be a disk-image
format: create a blank image, detect an existing one, convert it to raw, and
resize it.

!!! tip "Codecs do not import this module"
    Go's structural typing means any type whose method set is a superset of an
    interface satisfies it automatically. The [`qcow2`](qcow2.md) and
    [`dmg`](dmg.md) `Format` values satisfy `Format` here **without importing
    this module**. Only callers that need to use formats *polymorphically* import
    it and declare the interface variable.

## The interfaces

The full `Format` is composed from four single-purpose interfaces:

```go
// Creator can create a new blank disk image.
type Creator interface {
    Create(path string, sizeBytes int64) error
}

// Detector probes whether the file at path is in this format.
//   (true,  nil)  → valid image of this format
//   (false, nil)  → exists but is not this format
//   (false, err)  → path cannot be examined
type Detector interface {
    Detect(path string) (bool, error)
}

// RawConverter extracts an image to a plain raw disk image.
// src may be a file path or a directory (e.g. an OCI layout cache).
// Progress messages are written to w.
type RawConverter interface {
    ToRaw(src, dst string, w io.Writer) error
}

// Resizer adjusts the virtual size of an existing image. Growing must be
// supported; shrinking is optional and may return an error.
type Resizer interface {
    Resize(path string, newSizeBytes int64) error
}

// Format is the full contract: name + the four capabilities above.
type Format interface {
    Name() string   // e.g. "raw", "qcow2", "dmg"
    Creator
    Detector
    RawConverter
    Resizer
}
```

A consumer that only needs one capability can accept the narrow interface (e.g.
take a `Detector` to sniff a path) and stay decoupled from any specific codec.
