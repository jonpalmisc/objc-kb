# Objective-C ABI Knowledge Base

_**2023 Update:** I am also not actively working on projects related to
Objective-C anymore, and this document has not been updated in quite some
time. Some of the terminology here, etc. has always been wrong; other parts
have aged out since it was written. Some portions of this document remain
accurate, but in general, it would be wise to verify any claims made below._

---

This repository is meant to serve as a continuously-growing knowledge base
regarding the Objective-C ABI. It is a **work in progress** and is currently
just my personal observations and notes. It is likely that there are errors
and/or misunderstandings. Pull requests with corrections or new knowledge are
encouraged.

NOTE: In examples where structures/types are shown, some names have been
modified for clarity. Structure and member names shown do not always match
Objective-C's source code or their canonical names.

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

```c
100096d60 struct class_ro_t ro__TtC7Console21ReportsViewController = {
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

This section details structures used by the Objective-C ABI. The structure
layouts shown in the following subsections represent how structures are laid out
_at rest_, i.e. in a binary, not necessarily how they are laid out in memory at
runtime.

### Classes

**Class** structures are defined in the `__objc_data` section and are the basis
for how classes are stored inside the binary. Class structures have the
following layout:

```c
struct class_t {
    const void* isa;
    const class_t* super;       /* Superclass' `class_t` structure */
    void* cache;                /* Commonly `nullptr` */
    void* vtable;               /* Commonly `nullptr` */
    const class_ro_t* data;     /* Associated class RO structure */
};
```

> Any of the members which are pointers may be tagged or image-relative. The
> `data` member may be a fast pointer.

### Class RO

**Class RO** (read-only) structures are defined inside of the `__objc_const`
section. Most of the "interesting" information about classes&mdash;such as name,
methods, or instance variables&mdash;is stored in class RO structures. The
layout of class RO structures is as follows:

```c
struct class_ro_t {
    uint32_t flags;             /* Flags */
    uint32_t start;
    uint32_t size;
    uint32_t reserved;          /* Reserved for future use */
    const void* ivar_layout;
    const char* name;             /* Class name */
    const method_list_t* methods; /* Base method list */
    const void* protocols;
    const void* vars;
    const void* weak_ivar_layout;
    const void* properties;
};
```

> Any of the members which are pointers may be tagged or image-relative.

### Method lists

**Method lists** are found in the `__objc_const` section on x86_64, or under
`__objc_methlist` section under arm64(e). Method lists describe all of the base
methods associated with a class.

#### Header

A method list begins with a **method list header**, which has the following
format:

```c
struct method_list_t {
    uint32_t size_and_flags;    /* Entry size and flags */
    uint32_t count;             /* Number of entries */
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

Immediately following the method list's header comes one or more **entries**.
Entries may have either of the following layouts:

```c
struct method_t {
    const char* name;           /* Pointer to name (or selector reference?) */
    const char* types;          /* Pointer to type info */
    void* imp;                  /* Pointer to implementation (code) */
};

struct method_entry_t {
    int32_t name;               /* Relative offset to name or selector reference */
    int32_t types;              /* Relative offset to type info */
    int32_t imp;                /* Relative offset to implementation (code) */
};
```

The former entry layout (hereafter the "legacy" format) is older and is
primarily used on x86_64. The latter format (hereafter the "modern" format) is
newer and is used on arm64(e).

The modern format always utilizes **relative offsets**&mdash;not to be confused
with image-relative pointers&mdash;to point to its associated data. These
offsets are to be interpreted as offsets from the structure member's absolute
position in memory.

To illustrate the difference between the two formats, have a look at the method
list for the `BitFieldBox` in macOS 12's Calculator.app. Below is an excerpt of
the method list from the x86_64 slice, which uses the legacy format:

```c
100028198 struct method_list_t ml_BitFieldBox = {
100028198     uint32_t size_and_flags = 0x18 /* No flags; 24-byte entries */
10002819c     uint32_t count = 0x4           /* 4 methods (only 1 shown here) */
1000281a0 }
1000281a0 struct method_t mt_initWithFrame_ = {
1000281a0     const char* name = 0x10001c6e8  /* &"initWithFrame:" */
1000281a8     const char* types = 0x1000214c5 /* &"@48@0:8{CGRect={CGPoint=dd}{CGSize=dd}}16" */
1000281b0     void* imp = 0x100006829	      /* [BitFieldBox initWithFrame:] */
1000281b8 }
```

In comparison, here is an excerpt of the same method list in the arm64e slice of
the binary:

```c
10001f3b0  struct method_list_t ml_BitFieldBox = {
10001f3b0      uint32_t size_and_flags = 0x8000000c /* HAS_RELATIVE_OFFSETS; 12-byte entries */
10001f3b4      uint32_t count = 0x4                 /* 4 entries (only 1 shown here) */
10001f3b8  }
10001f3b8  struct method_entry_t mt_initWithFrame_ = {
10001f3b8      int32_t name = 0x12498 /* 0x100031850 = &&"initWithFrame:" */
10001f3bc      int32_t types = 0x6165 /* 0x100025521 = &"@48@0:8{CGRect={CGPoint=dd}{CGSize=dd}}16" */
10001f3c0      int32_t imp = -0x16d34 /* 0x10000868c = [BitFieldBox initWithFrame:] */
10001f3c4  }
```


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
