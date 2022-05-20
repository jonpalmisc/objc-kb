# Objective-C ABI Knowledge Base

This repository is meant to serve as a continuously-growing knowledge base
regarding the Objective-C ABI. It is a **work in progress** and is currently
just my personal observations and notes. It is likely that there are errors
and/or misunderstandings. Pull requests with corrections or new knowledge are
encouraged.

NOTE: In examples where structures/types are shown, some fields have been
renamed to more clearly indicate their purpose; field names shown do not always
match Objective-C's source code or their canonical names.

## Pointer types

The Objective-C ABI utilizes numerous different encoding techniques for
pointers. Different techniques are used depending on the part of the ABI in
question, the architecture of the binary, and the OS version the binary was
compiled for. Objective-C's internal structures make heavy use of (wacky)
pointers, so understanding different pointer types is important before looking
at structures in detail.

The following table is a **rough outline** of how pointers are typically
encoded, given an OS version and architecture:

| OS        | Architecture | Tagged | Type           |
|-----------|--------------|--------|----------------|
| macOS     | x86_64       | No     | Absolute       |
| macOS 11  | arm64        | Yes    | Absolute       |
| macOS 12+ | arm64e       | Ye     | Image-relative |

The table above is **not** all-encompassing and ignores certain scenarios; use
this for a general understanding, not for writing tools.

### Absolute (C-style) pointers

Absolute (C-style) pointers are sometimes used, but are becoming less common.
They look like "normal" pointers (shown below) and don't have any quirks. As of
2022, absolute pointers are mostly only still used on x86_64.

```
0x1000311a0
0x100031240
```

### Image-relative pointers

**Image-relative pointers** are really offsets relative to the image base.
Assuming a standard Mach-O image base of 4 GB (`0x100000000`), the
image-relative pointer `0x25207` is equivalent to `0x100025207`. Image-relative
pointers can be combined with some of the other pointer types detailed below,
e.g. a tagged pointer.

### Tagged pointers

**Tagged pointers**&mdash;used in arm64(e) binaries&mdash;pointers that carry
metadata with them in their upper bits. They look like the following:

```
0x800d6ae1000331c8
0x800000002c1d8
```

The metadata portion of the pointer can be removed by performing a bitwise AND
against `0x7ffffffff`.

```
0x800d6ae1000331c8 & 0x7ffffffff = 0x1000331c8
0x800000002c1d8 & 0x7ffffffff = 0x2c1d8
```

This illustrates an important point: even after removing the metadata, the
resulting pointer can be **absolute** or **image-relative**.

### Fast pointers

_WARNING: This section is likely incomplete._

**Fast pointers** are a type of pointer often used by Swift's ABI, which
overlaps Objective-C's ABI. Fast pointers store metadata in the two least
significant bits of the pointer. These two bits should be removed when
attempting to dereference the pointer.

Here is an example of a fast pointer, taken from the arm64e slice of the main
binary of macOS 12's "Console.app":

```
0x20000000096d62
```

The pointer is fast, tagged, and relative. After removing the tags and adding
the image base, the resulting pointer is produced:

```
0x100096d62
```

However, if we dereferenced this pointer as is, we would be (incorrectly)
pointing into the middle of the following structure:

```
100096d60 struct objc_class_ro_t ro__TtC7Console21ReportsViewController =
100096d60 {
100096d60     uint32_t flags = 0x184
100096d64     uint32_t start = 0x48
100096d68     uint32_t size = 0x68
100096d6c     uint32_t reserved = 0x0
100096d70     void* ivar_layout = NULL
100096d78     void* name = nm__TtC7Console21ReportsViewController
100096d80     void* methods = ml__TtC7Console21ReportsViewController
100096d88     void* protocols = NULL
100096d90     void* vars = 0x10008c068
100096d98     void* weak_ivar_layout = NULL
100096da0     void* properties = 0x10008c0f0
100096da8 }
```

> The definition of this strucutre isn't really important here, but is used to
> illustrate an example.

After removing the flags in the two least significant bits, we get the correct
pointer, which points to the structure's base:

```
0x100096d62 & (~0b11) = 0x100096d60
```

## Types and structures

Coming soon.

### Class metadata

Coming soon.

### Classes

Coming soon.

### Method lists

**Method lists** are found in the `__objc_const` section on x86_64, or under
`__objc_methlist` section under arm64(e). Method lists describe all of the base
methods associated with a class.

#### Header

A method list begins with a **method list header**, which has the following
format:

```c
struct objc_method_list_t {
    uint32_t size_and_flags;
    uint32_t count;
};
```

The `size_and_flags` field tells the size of the each entry, and optionally has
flags in high bits. Flags can be isolated by performing a bitwise AND of the
`size_and_flags` field with `0xffff0000`. The following flags may be present in
the `size_and_flags` field:

| Flag                   | Value        |
|------------------------|--------------|
| `HAS_RELATIVE_OFFSETS` | `0x80000000` |
| `HAS_DIRECT_SELECTORS` | `0x40000000` |

The `HAS_RELATIVE_OFFSETS` flag tells whether the pointers in the method list's
entries should be treated as absolute pointers or relative offsets (explained in
more detail below).

The `HAS_DIRECT_SELECTORS` flag tells what the name/selector field in this
method lists's entries points to. If the flag is set, the field points directly
to a string; if the flag is unset, it points to a selector reference.

#### Entries

After a method list's header comes one or more **entries**. Entries can come in
either of the following forms:

```c
struct objc_method_t {
    void* name;
    void* types;
    void* imp;
};

struct objc_method_entry_t {
    int32_t name;
    int32_t types;
    int32_t imp;
};
```

The former entry layout (hereafter the "legacy" format) is older and is
primarily used on x86_64. The latter format (hereafter the "modern" format) is
newer and is used on arm64(e).

The modern format always utilizes **relative offsets**&mdash;not to be confused
with image-relative pointers&mdash;to point to its associated data. These
offsets are to be interpreted as offsets from the structure member's absolute
position in memory.

## Further reading

Below are some resources and projects related to the Objective-C ABI that may be
useful references.

**Resources**

- https://developpaper.com/in-depth-analysis-of-the-structure-of-the-method-in-objc/
- https://www.fortinet.com/blog/threat-research/rewriting-idapython-script-objc2-xrefs-helper-py-for-hopper

**Projects**

- https://github.com/cxnder/ktool
- https://github.com/blacktop/ipsw
- https://github.com/jonpalmisc/ObjectiveNinja
