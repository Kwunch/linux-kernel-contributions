# rust: list: fix incomplete SAFETY comments in list implementation

**Status:** Under Review  
**Submitted:** April 3, 2026  
**Mailing list:** [lore.kernel.org](https://lore.kernel.org/rust-for-linux/) *(search "Christian Benton" for thread)*  
**Subsystem:** Rust / Linked List  
**Series:** [PATCH 0/2]  

## Overview

Four `// SAFETY: TODO.` comments were left incomplete in the Linux kernel's
intrusive linked list implementation. These placeholders indicate unsafe operations
whose correctness had not yet been formally documented. This patch series fills
them in with proper safety proofs.

Writing correct SAFETY comments is critical in Rust kernel code — they are the
human-written proof that an `unsafe` block is actually sound. An incomplete or
incorrect SAFETY comment is worse than none at all, as it could mask real bugs
during future code review.

## Files Changed

| File | Change |
|------|--------|
| `rust/kernel/list.rs` | Fixed 1 SAFETY comment in `List::remove` |
| `rust/kernel/list/impl_list_item_mod.rs` | Fixed 3 SAFETY comments in `impl_has_list_links_self_ptr!` macro |

## Patches

### [PATCH 1/2] rust: list: fix SAFETY comment in List::remove

**File:** `rust/kernel/list.rs`  
**Location:** `List::remove()` method  

**Before:**
```rust
// SAFETY: TODO.
let mut item = unsafe { ListLinks::fields(T::view_links(item)) };
```

**After:**
```rust
// SAFETY: `T::view_links` returns a reference to the `ListLinks` field of `item`.
// References are always valid and non-dangling, so converting to a raw pointer
// via `ListLinks::fields` is sound.
let mut item = unsafe { ListLinks::fields(T::view_links(item)) };
```

**Reasoning:** `T::view_links` returns a Rust reference, which by definition is
always valid and non-dangling. Converting that reference to a raw pointer for
`ListLinks::fields` is therefore sound.

---

### [PATCH 2/2] rust: list: fix SAFETY comments in impl_list_item_mod

**File:** `rust/kernel/list/impl_list_item_mod.rs`  
**Location:** `impl_has_list_links_self_ptr!` macro — three unsafe blocks  

#### Fix 1 — `HasListLinks` implementation

**Before:**
```rust
// SAFETY: TODO.
unsafe impl HasListLinks for $self { ... }
```

**After:**
```rust
// SAFETY: The implementation of `raw_get_list_links` returns a pointer to the
// `ListLinks` field inside `ListLinksSelfPtr`. This cast is valid because
// `ListLinksSelfPtr` is a wrapper around `ListLinks` and shares the same memory
// layout. The macro only compiles if the field has type `ListLinksSelfPtr`, which
// the type system enforces statically.
```

**Reasoning:** The type system enforces correctness — the macro only compiles if
the field actually has type `ListLinksSelfPtr`. The cast from `ListLinksSelfPtr`
to `ListLinks` is valid because `ListLinksSelfPtr` wraps `ListLinks` with the
same memory layout.

---

#### Fix 2 — `prepare_to_insert`

**Before:**
```rust
// SAFETY: TODO.
let container = unsafe { container_of!(links_field, ListLinksSelfPtr<Self, $num>, inner) };
```

**After:**
```rust
// SAFETY: `links_field` is valid because `view_links` returned it from a valid
// `me` pointer as promised by the caller. `links_field` points to the `inner`
// field of a `ListLinksSelfPtr` because `Self: HasSelfPtr` guarantees that the
// `ListLinks` field is always inside a `ListLinksSelfPtr`.
```

**Reasoning:** Two invariants make this safe — `links_field` is valid because it
comes from `view_links` applied to a caller-guaranteed valid pointer, and
`Self: HasSelfPtr` statically guarantees that the `ListLinks` field lives inside
a `ListLinksSelfPtr`, making the `container_of!` offset correct.

---

#### Fix 3 — `view_value`

**Before:**
```rust
// SAFETY: TODO.
let container = unsafe { container_of!(links_field, ListLinksSelfPtr<Self, $num>, inner) };
```

**After:**
```rust
// SAFETY: `links_field` is valid and points to a live value because the caller
// of `prepare_to_insert` promised to retain ownership of the `ListArc`, and the
// value cannot be destroyed while a `ListArc` exists. `links_field` points to
// the `inner` field of a `ListLinksSelfPtr` because `Self: HasSelfPtr`
// guarantees this.
```

**Reasoning:** The `ListArc` ownership guarantee from `prepare_to_insert`'s
caller ensures the value is still alive. `Self: HasSelfPtr` guarantees the
`container_of!` offset is correct.

## What is a SAFETY Comment?

In Rust, `unsafe` blocks require the programmer to manually uphold invariants
the compiler cannot verify. A `// SAFETY:` comment is the standard way to
document why an `unsafe` operation is actually sound. In the Linux kernel's Rust
code, these comments are required and reviewed carefully — an incomplete comment
flagged as `TODO` represents a gap in the formal safety documentation.

## Patch Files

- [`0000-cover-letter.patch`](./0000-cover-letter.patch)
- [`0001-rust-list-fix-SAFETY-comment-in-List-remove.patch`](./0001-rust-list-fix-SAFETY-comment-in-List-remove.patch)
- [`0002-rust-list-fix-SAFETY-comments-in-impl_list_item_mod.patch`](./0002-rust-list-fix-SAFETY-comments-in-impl_list_item_mod.patch)

## Related

- [Rust-for-Linux project](https://rust-for-linux.com/)
- [Linux kernel unsafe code guidelines](https://docs.kernel.org/rust/coding-guidelines.html)
- [stream_open contribution](../stream_open/README.md)
