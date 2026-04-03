# rust: seq_file: add puts, putc, write, and hex_dump methods

**Status:** Under Review  
**Submitted:** April 3, 2026  
**Mailing list:** [lore.kernel.org](https://lore.kernel.org/rust-for-linux/) *(search "Christian Benton" for thread)*  
**Subsystem:** Rust / Seq File  

## Overview

The `SeqFile` abstraction in the Linux kernel Rust API currently only exposes
`seq_printf` via the `seq_print!` macro. This leaves several commonly used
seq_file operations unavailable to Rust kernel code, requiring drivers to either
use awkward formatted output for everything or call through unsafe bindings
directly.

This patch adds four new methods to `SeqFile` covering the most commonly used
seq_file operations beyond printf.

## What is seq_file?

`seq_file` is a kernel abstraction for generating text output to virtual files —
primarily `/proc` and `/debugfs` entries. It handles buffering, pagination, and
the complexity of userspace reading output in chunks. Rust drivers implementing
interfaces like `show_fdinfo` in `MiscDevice` use `SeqFile` to write their output.

## Files Changed

| File | Change |
|------|--------|
| `rust/kernel/seq_file.rs` | Added `puts`, `putc`, `write`, `hex_dump` methods and `HexDumpPrefix` enum |

## Methods Added

### `puts(s: &CStr)`

Writes a C string to the seq file. Wraps `__seq_puts()` — the non-inline version
of `seq_puts()` which is accessible through the kernel bindings.
```rust
pub fn puts(&self, s: &CStr) {
    // SAFETY: `self.inner.get()` is valid because `&self` guarantees the
    // `SeqFile` is alive and was properly initialized via `from_raw`.
    // `s.as_char_ptr()` is valid because `CStr` is always a valid
    // null-terminated C string.
    unsafe { bindings::__seq_puts(self.inner.get(), s.as_char_ptr()) }
}
```

### `putc(c: u8)`

Writes a single byte to the seq file. Wraps `seq_putc()`.
```rust
pub fn putc(&self, c: u8) {
    // SAFETY: `self.inner.get()` is valid because `&self` guarantees the
    // `SeqFile` is alive and was properly initialized via `from_raw`.
    unsafe { bindings::seq_putc(self.inner.get(), c as ffi::c_char) }
}
```

### `write(data: &[u8])`

Writes raw bytes to the seq file. Wraps `seq_write()`. Takes a `&[u8]` slice
rather than a separate pointer and length, ensuring they can never be mismatched.
```rust
pub fn write(&self, data: &[u8]) {
    // SAFETY: `self.inner.get()` is valid because `&self` guarantees the
    // `SeqFile` is alive and was properly initialized via `from_raw`.
    // `data.as_ptr()` is valid and non-dangling because it comes from a
    // `&[u8]`, which guarantees the memory is valid for `data.len()` bytes
    // and will not be modified during the call due to the shared reference.
    unsafe {
        bindings::seq_write(
            self.inner.get(),
            data.as_ptr().cast::<ffi::c_void>(),
            data.len(),
        )
    };
}
```

### `hex_dump(...)`

Dumps binary data as formatted hex output. Wraps `seq_hex_dump()`.
```rust
pub fn hex_dump(
    &self,
    prefix_str: &CStr,
    prefix_type: HexDumpPrefix,
    rowsize: u8,
    groupsize: u8,
    buf: &[u8],
    ascii: bool,
)
```

## HexDumpPrefix Enum

The C `seq_hex_dump()` function takes a raw `int` for the prefix type with magic
values 0, 1, and 2. This patch replaces that with a proper Rust enum, making
invalid values unrepresentable at compile time.
```rust
pub enum HexDumpPrefix {
    /// No prefix.
    None,
    /// Prefix with the memory address.
    Address,
    /// Prefix with the offset within the buffer.
    Offset,
}
```

This follows the Rust principle of making invalid states unrepresentable — a
caller cannot accidentally pass an invalid prefix type because the type system
only allows three variants.

## Why These Methods Are Needed

Rust drivers implementing `show_fdinfo` in `MiscDevice` receive a `&SeqFile` and
must write formatted output to it. Currently the only available method is
`seq_print!` which requires format strings for all output. Simple operations like
writing a fixed string or a byte require going through the formatting machinery
unnecessarily, or calling into unsafe bindings directly.

These four methods cover the most commonly used seq_file operations and give Rust
drivers a complete, safe API for generating seq_file output.

## Patch File

The formatted patch ready for submission is available at
[`0001-rust-seq_file-add-puts-putc-write-and-hex_dump-metho.patch`](./0001-rust-seq_file-add-puts-putc-write-and-hex_dump-metho.patch).

## Related

- [Rust-for-Linux project](https://rust-for-linux.com/)
- [seq_file kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/seq_file.html)
- [stream_open contribution](../stream_open/README.md)
- [list_safety contribution](../list_safety/README.md)
