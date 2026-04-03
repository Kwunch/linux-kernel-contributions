# rust: fs: file: add FMODE constants and stream_open for LocalFile

**Status:** Under Review  
**Submitted:** April 3, 2026  
**Mailing list:** [lore.kernel.org](https://lore.kernel.org/rust-for-linux/20260403201941.37501-1-t1bur0n.kernel.org@protonmail.ch/T/#u)  
**Subsystem:** Rust / File System  

## Overview

The C function `stream_open()` had no Rust equivalent, requiring Rust drivers to
manipulate `f_mode` directly through unsafe bindings. Additionally, the five file
mode constants used by `stream_open()` were inaccessible to Rust kernel code due
to `bindgen` being unable to handle the `__force` sparse annotation used in their
C macro definitions.

This patch fills both gaps.

## Files Changed

| File | Change |
|------|--------|
| `rust/bindings/bindings_helper.h` | Added 5 FMODE constants |
| `rust/kernel/fs/file.rs` | Added `stream_open()` to `impl LocalFile` |

## Constants Added

Added to `rust/bindings/bindings_helper.h` using the `RUST_CONST_HELPER` pattern:

| Constant | Value | Description |
|----------|-------|-------------|
| `FMODE_STREAM` | `1 << 21` | Marks a file as a stream, disabling seek position validation |
| `FMODE_LSEEK` | `1 << 2` | Indicates the file supports seeking |
| `FMODE_PREAD` | `1 << 3` | Indicates the file supports positional reads |
| `FMODE_PWRITE` | `1 << 4` | Indicates the file supports positional writes |
| `FMODE_ATOMIC_POS` | `1 << 15` | Indicates atomic file position updates |

## stream_open() Implementation

Added to `impl LocalFile` in `rust/kernel/fs/file.rs`:
```rust
/// Marks this file as a stream, disabling seek position validation.
///
/// This should be called from the `open` handler of a character device that
/// produces data as a stream rather than supporting random access. It clears
/// the seek-related file mode flags and sets `FMODE_STREAM`, which tells the
/// VFS layer not to validate seek positions on `read_iter` and `write_iter`
/// calls.
///
/// Must only be called during `open()`, before the file descriptor is
/// installed into the process fd table and becomes accessible to other
/// threads.
pub fn stream_open(&self) {
    // SAFETY: The file pointer is valid for the duration of this call
    // because `LocalFile` can only exist while the underlying `struct file`
    // is alive. Since `LocalFile` is not `Send`, this method can only be
    // called before the file is shared across threads, which means no other
    // thread can be concurrently modifying `f_mode`. The pointer is
    // guaranteed non-null by the type invariants of `LocalFile`.
    unsafe {
        let file = self.as_ptr();
        (*file).f_mode &= !(
            bindings::FMODE_LSEEK
                | bindings::FMODE_PREAD
                | bindings::FMODE_PWRITE
                | bindings::FMODE_ATOMIC_POS
        );
        (*file).f_mode |= bindings::FMODE_STREAM;
    }
}
```

## Design Decisions

**Why `LocalFile` and not `File`?**

`stream_open()` must be called during `open()`, before the file descriptor is
installed into the process fd table. `LocalFile` is `!Send`, which statically
guarantees that no other thread can concurrently access `f_mode` at the point
this method is called. Placing the method on `File` instead would allow calling
it at incorrect times with no compile-time protection.

**Why not call `bindings::stream_open()` directly?**

The C `stream_open()` function takes an inode parameter that it never uses. Calling
it from Rust would require obtaining a dummy inode pointer unnecessarily. Since the
function only manipulates `f_mode`, reimplementing it directly in Rust using the
newly added FMODE constants is cleaner and eliminates the unnecessary C call.

**Why not use `write_volatile` for `f_mode`?**

The `LocalFile` type guarantees exclusive access to `f_mode` at the point
`stream_open()` is called, making `write_volatile` unnecessary. Regular assignment
through a raw pointer is sufficient and more readable.

## C Reference Implementation

The C function this patch replaces (`fs/open.c`):
```c
int stream_open(struct inode *inode, struct file *filp)
{
    filp->f_mode &= ~(FMODE_LSEEK | FMODE_PREAD | FMODE_PWRITE | FMODE_ATOMIC_POS);
    filp->f_mode |= FMODE_STREAM;
    return 0;
}
EXPORT_SYMBOL(stream_open);
```

Note that the `inode` parameter is unused. The Rust implementation omits it entirely.

## Patch File

The formatted patch ready for submission is available at
[`0001-rust-fs-file-add-FMODE-constants-and-stream_open-for.patch`](./0001-rust-fs-file-add-FMODE-constants-and-stream_open-for.patch).

## Related

- [Rust-for-Linux project](https://rust-for-linux.com/)
- [stream_open() kernel docs](https://www.kernel.org/doc/html/latest/filesystems/locking.html)
- [bindgen __force issue](https://github.com/rust-lang/rust-bindgen/issues/3179)
