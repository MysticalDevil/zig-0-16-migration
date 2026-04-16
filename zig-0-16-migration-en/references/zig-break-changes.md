# Zig 0.13 → 0.16 Breaking Changes Quick Reference

> This document compiles all breaking changes to watch for when upgrading from Zig 0.13 to 0.16, verified against the local `zig 0.16.0` compiler.

---

## Quick Reference

| Old syntax (≤0.13) | 0.16 correct syntax | Category |
|---|---|---|
| `@setCold(true)` | `@branchHint(.cold)` | Language |
| `@fence(.seq_cst)` | Atomic ops / `asm volatile` | Language |
| `@Type(.{ .int = ... })` | `@Int(...)` | Language |
| `@Type(.{ .@"struct" = .{ .is_tuple = true, ... } })` | `@Tuple(&.{ ... })` | Language |
| `@Type(...)` (other types) | `@Pointer` / `@Fn` / `@Struct` / `@Union` / `@Enum` / `@EnumLiteral` | Language |
| `@intFromFloat(x)` | `@trunc(x)` / `@floor(x)` etc. | Language |
| `@export(x, .{ ... })` | `@export(&x, .{ ... })` | Language |
| `usingnamespace @import("std");` | Explicit imports / `switch` / `inline for` | Language |
| `async foo()` / `await frame` | Rewrite as synchronous or use `std.Io` | Language |
| `switch` on non-exhaustive enum | Must add `else` | Language |
| `@ptrCast(&arr)` array→vector | In-memory coercion forbidden | Language |
| `?*const anyopaque` in `@typeInfo` | Type expression simplified | Language |
| `std.ArrayList(u8).init(allocator)` | `var arr: std.ArrayList(u8) = .empty;` | std |
| `std.ArrayListManaged(u8)` | Still works but deprecated | std |
| `std.BoundedArray(T, n)` | ❌ Deleted | std |
| `std.heap.GeneralPurposeAllocator` | `std.heap.DebugAllocator` | std |
| `std.Thread.Pool` | `std.Io.Group` | std |
| `std.heap.ThreadSafeAllocator` | `std.heap.ArenaAllocator` | std |
| `std.io.GenericReader` / `AnyReader` / `FixedBufferStream` | ❌ Deleted | std |
| `std.fs.cwd().createFile(...)` | `std.Io.Dir.cwd().createFile(io, ...)` | std |
| `std.process.args` / `std.process.env` | `main(init: std.process.Init)` | std |
| `std.time.Instant.now()` | `std.Io.Timestamp.now(io, .real)` | std |
| `std.crypto.random.bytes(&buf)` | `io.random(&buf)` | std |
| `std.builtin.subsystem` | `std.zig.Subsystem` | std |
| `std.fs.getAppDataDir` | ❌ Deleted | std |
| `b.addExecutable(.{ .root_source_file = ... })` | `b.createModule(...)` + `b.addExecutable(.{ .root_module = ... })` | Build |
| `exe.linkLibC()` | `root_mod.link_libc = true` | Build |

---

## 1. Language

### 1.1 Removed builtins

#### `@setCold` → `@branchHint`

```zig
// ≤0.13
@setCold(true);

// 0.16
@branchHint(.cold);
```

**Verified (0.16)**: `error: invalid builtin function: '@setCold'`

#### `@fence` → atomic ops / `asm volatile`

```zig
// ≤0.13
@fence(.seq_cst);
```

**Verified (0.16)**: `error: invalid builtin function: '@fence'`

#### `@Type` split into 8 independent builtins

```zig
// ≤0.15
const T = @Type(.{ .int = .{ .signedness = .unsigned, .bits = 8 } });

// 0.16
const T = @Int(.unsigned, 8);
```

**Verified (0.16)**: `error: invalid builtin function: '@Type'`

Full mapping:

| Old `@Type` | 0.16 builtin |
|---|---|
| `@Type(.{ .int = ... })` | `@Int(...)` |
| `@Type(.{ .@"struct" = .{ .is_tuple = true, ... } })` | `@Tuple(&.{ ... })` |
| `@Type(.{ .pointer = ... })` | `@Pointer(...)` |
| `@Type(.{ .@"fn" = ... })` | `@Fn(...)` |
| `@Type(.{ .@"struct" = ... })` | `@Struct(...)` |
| `@Type(.{ .@"union" = ... })` | `@Union(...)` |
| `@Type(.{ .@"enum" = ... })` | `@Enum(...)` |
| `@Type(.enum_literal)` | `@EnumLiteral()` |

#### `@intFromFloat` deprecated

Still works, but prefer `@trunc` / `@floor` / `@ceil` / `@round` for direct integer conversion.

```zig
const x: f32 = 3.7;
const y: u32 = @trunc(x); // 0.16 allows, result is 3
```

### 1.2 Syntax removal

#### `usingnamespace` fully removed

```zig
// ≤0.14
usingnamespace @import("std");
```

**Verified (0.16)**: `error: expected ',' after field`

**Alternatives**:
- Conditional inclusion: use `if` or comptime `switch`
- Implementation switching: use `inline for` or comptime selection
- Mixins: explicitly import fields/functions into current namespace

#### `async` / `await` keywords removed

```zig
// ≤0.14
const frame = async foo();
await frame;
```

**Verified (0.16)**: `error: expected ';' after statement`

**Alternative**: Rewrite as synchronous code, or use `std.Io.Future` / `std.Io.Group`.

### 1.3 Semantic tightening

#### `@export` operand must be comptime-known pointer

```zig
export var x: u32 = 0;

// ≤0.13 allowed value export; 0.14+ errors
@export(x, .{ .name = "y" }); // error: export target must be comptime-known

// 0.16 correct syntax
@export(&x, .{ .name = "y" });
```

#### Sentinel must be scalar type

```zig
// 0.16 error: expected type '[2]u8', found 'comptime_int'
var x: [*:0]const [2]u8 = undefined;
```

#### In-memory coercion forbidden

```zig
var arr: [4]u8 = .{ 1, 2, 3, 4 };
const vec: @Vector(4, u8) = @ptrCast(&arr); // 0.16 error
```

**Verified (0.16)**: `error: expected pointer type, found '@Vector(4, u8)'`

#### `switch` on non-exhaustive enum must have `else`

```zig
const E = enum(u8) { a, b, _ };
fn foo(e: E) void {
    switch (e) {
        .a => {},
        .b => {},
        else => {}, // required in 0.15+
    }
}
```

#### Returning address of local variable from function forbidden

```zig
fn foo() *u32 {
    var x: u32 = 42;
    return &x; // 0.16 compile error
}
```

#### Runtime vector index forbidden

```zig
fn bar(i: usize) u32 {
    const v = @Vector(4, u32){ 1, 2, 3, 4 };
    return v[i]; // 0.16 error
}
```

#### packed union / packed struct rules tightened

- packed unions cannot contain pointer fields
- packed unions require all field bit widths to match backing integer (unused bits forbidden)
- extern context no longer allows implicit backing type for enum / packed types
- packed unions support explicit backing integer: `packed union(u32) { ... }`

#### Explicitly-aligned pointers and naturally-aligned pointers are distinct types

```zig
var x: u32 = 0;
const p1: *u32 = &x;
const p2: *align(4) u32 = &x;
// In 0.16 p1 and p2 are no longer the same type
```

#### Small integers can implicitly coerce to float

```zig
const x: u24 = 123;
const y: f32 = x; // 0.16 allows (lossless)
```

### 1.4 Type system adjustments

#### `std.builtin.Type` fields renamed

| Old field name | New field name |
|---|---|
| `.Struct` | `.@"struct"` |
| `.Union` | `.@"union"` |
| `.Enum` | `.@"enum"` |
| `.Fn` | `.@"fn"` |
| `.Pointer.size` `.One` | `.one` (lowercase) |

Impact: all metaprogramming code relying on `@typeInfo` + `std.builtin.Type` needs updated field access.

#### `?*const anyopaque` simplified in `@typeInfo`

Since 0.14, `std.builtin.Type.Pointer` `child` handling was simplified. Many scenarios that previously required explicit `?*const anyopaque` can now use more natural type expressions.

### 1.5 Other

#### `@cImport` marked deprecated

Still works (no deprecation warning), but official direction is to migrate fully to Build System's `b.addTranslateC`.

---

## 2. Standard Library

### 2.1 Allocators

| Old API | 0.16 status |
|---|---|
| `std.heap.GeneralPurposeAllocator` | Removed → use `std.heap.DebugAllocator` |
| `std.heap.SmpAllocator` | New, high-performance multi-threaded allocator |
| `std.heap.ThreadSafeAllocator` | Removed → use `std.heap.ArenaAllocator` (now thread-safe) |
| `Allocator` vtable | New `remap` field; custom allocators must implement it |

**Verified (0.16)**:

```bash
error: root source file struct 'heap' has no member named 'ThreadSafeAllocator'
```

### 2.2 Containers and data structures

#### `ArrayList` defaults to Unmanaged

```zig
// ≤0.14
var list = std.ArrayList(u8).init(allocator);

// 0.16 error: has no member named 'init'

// 0.16 correct syntax (Unmanaged)
var arr: std.ArrayList(u8) = .empty;
try arr.append(allocator, 'a');

// If you still need Managed (deprecated)
var list = std.ArrayListManaged(u8).init(allocator);
```

**Verified (0.16)**: `error: struct 'array_list.Aligned(u8,null)' has no member named 'init'`

#### `BoundedArray` deleted

```zig
// ≤0.14
var buf = std.BoundedArray(u8, 256).init(0) catch unreachable;
```

**0.16**: `std.BoundedArray` is deleted with no direct replacement.

#### Linked lists de-genericized

`std.SinglyLinkedList` / `std.DoublyLinkedList` removed many generic parameters; APIs are more fixed.

### 2.3 I/O overhaul (Writergate → `std.Io`)

In 0.16 `std.Io` becomes a first-class abstraction; many old APIs were migrated or deleted.

| Old API (≤0.15) | 0.16 status |
|---|---|
| `std.fs.cwd().createFile(...)` | `std.Io.Dir.cwd().createFile(io, ...)` |
| `std.fs.File.readAll(...)` | `std.Io.File.readPositionalAll(io, ...)` |
| `std.io.GenericReader` / `AnyReader` / `FixedBufferStream` | ❌ Deleted |
| `std.Thread.Pool` | ❌ Deleted → `std.Io.Group` |
| `std.time.Instant.now()` | `std.Io.Timestamp.now(io, .real)` |
| `std.crypto.random.bytes(&buf)` | `io.random(&buf)` |

**Verified (0.16)**:

```bash
error: root source file struct 'Thread' has no member named 'Pool'
error: root source file struct 'std' has no member named 'io'
```

### 2.4 Process and environment

| Old API | 0.16 status |
|---|---|
| `std.process.args` / `std.process.env` | ❌ Deleted → `main(init: std.process.Init)` |
| `std.process.Child.collectOutput` | API signature changed |

### 2.5 Other deletions

| Old API | 0.16 status |
|---|---|
| `std.builtin.subsystem` | ❌ Deleted → `std.zig.Subsystem` |
| `std.fs.getAppDataDir` | ❌ Deleted |
| `std.c` bindings | Large-scale reorganization; symbols moved to platform submodules |

**Verified (0.16)**:

```bash
error: root source file struct 'builtin' has no member named 'subsystem'
error: root source file struct 'fs' has no member named 'getAppDataDir'
```

### 2.6 Formatting

- Calling `format` must explicitly use `"{f}"` specifier
- `format` method no longer receives format string and options arguments
- Formatting no longer handles Unicode (no automatic `%s` recognition)
- New format specifiers added

### 2.7 Panic Interface

`@panic` underlying handler interface changed; custom panic handler signatures need updating.

---

## 3. Build System

### 3.1 Modularization refactor

#### Implicit root module removed

```zig
// ≤0.15
const exe = b.addExecutable(.{
    .name = "app",
    .root_source_file = b.path("src/main.zig"),
});

// 0.16
const root_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
});
const exe = b.addExecutable(.{
    .name = "app",
    .root_module = root_mod,
});
```

#### `addTest` uses `root_module`

```zig
const exe_tests = b.addTest(.{
    .root_module = exe.root_module,
});
```

#### `addModule` signature simplified

```zig
const lib_mod = b.addModule("mylib", .{
    .root_source_file = b.path("src/lib.zig"),
});
```

### 3.2 Libraries and linking

#### `linkLibC()` removed

```zig
// ≤0.15
exe.linkLibC();

// 0.16
const root_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .link_libc = true,
});
```

**Verified (0.16)**: `error: no member named 'linkLibC'`

#### `addLibrary` introduced

`b.addLibrary` is the new unified entry point for creating library artifacts.

### 3.3 Other

- `build.zig.zon` dependency hash algorithm changed; 0.13 hashes are unrecognized by 0.16
- New step types: `Fmt Step`, `WriteFile Step`, `RemoveDir Step`
- `zig init` became the standard way to create new projects

---

## 4. Compiler Verification Summary (0.16)

| Change | 0.16 verification |
|---|---|
| `@setCold` removed | ✅ `invalid builtin function: '@setCold'` |
| `@fence` removed | ✅ `invalid builtin function: '@fence'` |
| `@Type` removed | ✅ `invalid builtin function: '@Type'` |
| `@export` requires pointer | ✅ `export target must be comptime-known` |
| Non-scalar sentinel forbidden | ✅ `expected type '[2]u8', found 'comptime_int'` |
| In-memory coercion restricted | ✅ `@ptrCast` array→vector errors |
| `usingnamespace` removed | ✅ `expected ',' after field` |
| `async`/`await` keywords removed | ✅ `expected ';' after statement` |
| `ArrayList` default unmanaged | ✅ `has no member named 'init'` |
| `Thread.Pool` deleted | ✅ `has no member named 'Pool'` |
| `heap.ThreadSafeAllocator` deleted | ✅ `has no member named 'ThreadSafeAllocator'` |
| `builtin.subsystem` deleted | ✅ `has no member named 'subsystem'` |
| `fs.getAppDataDir` deleted | ✅ `has no member named 'getAppDataDir'` |
| `linkLibC()` deleted | ✅ `no member named 'linkLibC'` |

---

## 5. Migration Priority Recommendations

1. **Language-level breaks** (`@Type`, `usingnamespace`, `async`/`await`, `@setCold`, `@fence`)
2. **Standard library APIs** (`ArrayList` → unmanaged, `std.Io` migration, deleted `Thread.Pool`, etc.)
3. **Build system** (`root_module`, `link_libc`)

---

## References

- Zig 0.14.0 Release Notes: `https://ziglang.org/download/0.14.0/release-notes.html`
- Zig 0.15.1 Release Notes: `https://ziglang.org/download/0.15.1/release-notes.html`
- Zig 0.16.0 Release Notes: `https://ziglang.org/download/0.16.0/release-notes.html`
- Local Zig 0.16.0 compiler verification
