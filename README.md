# Minimizing V (vlang) Binary Size

This repository demonstrates how to minimize the size of a V binary.

V compiles to C and by default produces relatively large binaries due to the included runtime, libc linkage, and debug information. With the right flags and techniques, you can significantly reduce the size to tens or even single-digit kilobytes.

All examples are tested with a simple `hello.v`:

```v
fn main() {
	println('Hello, World!')
}
```

### Step 0: Compilation Without Optimizations

```shell
v hello.v
```

Size: ~235.4 KB

### Step 1: Basic Optimization

```shell
v -prod hello.v
```

`-prod` enables production mode:
- removes bounds checks and asserts,
- enables optimizations,
- eliminates dead code.

Size: ~187.9 KB

### Step 2: Optimize Generated Code for Size

Pass `-Os` (optimize for size) to the backend C compiler:

```shell
v -prod -cflags "-Os" hello.v
```

Size: ~152.6 KB

Further reduce by discarding unused sections:

```shell
v -prod -cflags "-Os -ffunction-sections -fdata-sections" -ldflags "-Wl,--gc-sections" hello.v
```

Size: ~96.7 KB

### Step 3: Strip Debug Information

Removes symbols and debug info—usually saves 20–30%.

```shell
strip -s hello
```

Size: ~79.5 KB from ~96.7 KB, ~119.8 KB from ~152.6 KB

### Step 4: Disable Unnecessary Runtime Features

```shell
v -prod -gc none -d no_segfault_handler -cflags "-Os" hello.v
strip -s hello
```

- `-gc none` — completely disables the garbage collector (safe if no dynamic allocations).
- `-d no_segfault_handler` — removes the segmentation fault handler (no stack trace on crash).

Size: ~26.7 KB before strip, ~22.2 KB with strip

### Step 5: Choose the Best Backend Compiler

Clang often produces slightly smaller code:

```shell
v -prod -gc none -d no_segfault_handler -cc clang -cflags "-Os" hello.v
strip -s hello
```

Size: ~15.2 KB before strip, ~14.1 with strip

### Step 6: Compress with UPX — Biggest Win

Install UPX first from repositories from upx.github.io.

```shell
v -prod -gc none -d no_segfault_handler -cc clang -cflags "-Os" -compress hello.v
```

`-compress` automatically runs `upx --best`.
Trade-off: slight startup delay due to decompression in memory.

Size: ~6.8 KB.

### Step 7: Static Build (Optional)

For a fully self-contained binary without external dependencies:

```shell
v -cc musl-gcc -prod -gc none -d no_segfault_handler -cflags "-Os" -compress hello.v
```

Convenient for distribution.

Size: ~14.7 KB without UPX, ~6.5 KB with UPX (`-compress`)

### Summary Table

|              Stage                                              | Approximate Size |
|:---------------------------------------------------------------:|:----------------:|
| Default build (v hello.v)                                       | 235.4 KB         |
| `-prod`                                                         | 187.9 KB         |
| `-prod -cflags "-Os"`                                           | 152.6 KB         |
| `-prod -cflags "-Os"` + `strip`                                 | 119.8 KB         |
| All flags + runtime disables                                    | 26.7 KB          |
| All flags + runtime disables + `strip`                          | 22.2 KB          |
| All flags + runtime disables + `-cc clang`                      | 15.2 KB          |
| All flags + runtime disables + `-cc clang` + `strip`            | 14.1 KB          |
| All flags + runtime disables + `-compress` (UPX)                | 6.8 KB           |
| All flags + runtime disables + Static Build                     | 14.7 KB          |
| All flags + runtime disables + Static Build + `strip`           | 13.8 KB          |
| All flags + runtime disables + Static Build + `-compress` (UPX) | 6.5 KB           |

### Recommended Full Command for Regular Code

```shell
v -prod -gc none -d no_segfault_handler -cc clang -cflags "-Os" -compress file.v
```

For projects, set flags once in your environment:

```shell
export VFLAGS="-prod -gc none -d no_segfault_handler -cc clang -cflags \"-Os\" -compress"
```

Then simply run `v .` or `v file.v`.

V prioritizes fast compilation and simplicity over record-breaking binary sizes. These techniques give compact executables suitable for most use cases.
