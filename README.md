# Objective-C ABI Knowldegebase

This repository is meant to serve as a continuously-growing knowledge base
regarding the Objective-C ABI. It is a **work in progress** and is currently
just my personal observations and notes. It is likely that there are errors
and/or misunderstandings. Pull requests with corrections or new knowledge are
encouraged.

## Pointer Types

The Objective-C ABI utilizes numerous different encoding techniques for
pointers.

### C-style (absolute) pointers

Regular, C-style pointers are sometimes used, but are becoming less common. They
look like the following, and are only commonly used on x86_64:

```
0x1000311a0
0x100031240
```

### Relative pointers

Relative pointers are really offsets relative to the image base. Assuming a
standard Mach-O image base of 4 GB (`0x100000000`), the relative pointer
`0x25207` is equivalent to `0x100025207`. Relative pointers can be combined with
some of the other pointer types detailed below.

### Tagged pointers

Tagged pointers&mdash;used in arm64(e) binaries&mdash;pointers that carry
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
resulting pointer can be **absolute** or **relative**.
