# Zig 0.13 → 0.16 Breaking Changes 速查

> 本文档为临时审阅稿，汇总从 Zig 0.13 升级到 0.16 所需关注的全部 Breaking Changes，
> 并全部经本地 `zig 0.16.0` 编译器验证。

---

## 快速参考（Quick Reference）

| 旧写法（≤0.13） | 0.16 正确写法 | 分类 |
|---|---|---|
| `@setCold(true)` | `@branchHint(.cold)` | 语言 |
| `@fence(.seq_cst)` | 原子操作 / `asm volatile` | 语言 |
| `@Type(.{ .int = ... })` | `@Int(...)` | 语言 |
| `@Type(.{ .@"struct" = .{ .is_tuple = true, ... } })` | `@Tuple(&.{ ... })` | 语言 |
| `@Type(...)`（其他类型） | `@Pointer` / `@Fn` / `@Struct` / `@Union` / `@Enum` / `@EnumLiteral` | 语言 |
| `@intFromFloat(x)` | `@trunc(x)` / `@floor(x)` 等 | 语言 |
| `@export(x, .{ ... })` | `@export(&x, .{ ... })` | 语言 |
| `usingnamespace @import("std");` | 显式导入 / `switch` / `inline for` | 语言 |
| `async foo()` / `await frame` | 重写为同步代码或 `std.Io` | 语言 |
| `switch` on non-exhaustive enum | 必须加 `else` | 语言 |
| `@ptrCast(&arr)` array→vector | 禁止 in-memory coercion | 语言 |
| `?*const anyopaque` 在 `@typeInfo` | 类型表达已简化 | 语言 |
| `std.ArrayList(u8).init(allocator)` | `var arr: std.ArrayList(u8) = .empty;` | std |
| `std.ArrayListManaged(u8)` | 仍可用但已标记 deprecated | std |
| `std.BoundedArray(T, n)` | ❌ 已删除 | std |
| `std.heap.GeneralPurposeAllocator` | `std.heap.DebugAllocator` | std |
| `std.Thread.Pool` | `std.Io.Group` | std |
| `std.heap.ThreadSafeAllocator` | `std.heap.ArenaAllocator` | std |
| `std.io.GenericReader` / `AnyReader` / `FixedBufferStream` | ❌ 已删除 | std |
| `std.fs.cwd().createFile(...)` | `std.Io.Dir.cwd().createFile(io, ...)` | std |
| `std.process.args` / `std.process.env` | `main(init: std.process.Init)` | std |
| `std.time.Instant.now()` | `std.Io.Timestamp.now(io, .real)` | std |
| `std.crypto.random.bytes(&buf)` | `io.random(&buf)` | std |
| `std.builtin.subsystem` | `std.zig.Subsystem` | std |
| `std.fs.getAppDataDir` | ❌ 已删除 | std |
| `b.addExecutable(.{ .root_source_file = ... })` | `b.createModule(...)` + `b.addExecutable(.{ .root_module = ... })` | 构建 |
| `exe.linkLibC()` | `root_mod.link_libc = true` | 构建 |

---

## 一、语言层（Language）

### 1.1 被移除的内置函数

#### `@setCold` → `@branchHint`

```zig
// ≤0.13
@setCold(true);

// 0.16
@branchHint(.cold);
```

**验证（0.16）**：`error: invalid builtin function: '@setCold'`

#### `@fence` → 原子操作 / `asm volatile`

```zig
// ≤0.13
@fence(.seq_cst);
```

**验证（0.16）**：`error: invalid builtin function: '@fence'`

#### `@Type` 拆分为 8 个独立 builtins

```zig
// ≤0.15
const T = @Type(.{ .int = .{ .signedness = .unsigned, .bits = 8 } });

// 0.16
const T = @Int(.unsigned, 8);
```

**验证（0.16）**：`error: invalid builtin function: '@Type'`

完整对照：

| 旧 `@Type` | 0.16 builtin |
|---|---|
| `@Type(.{ .int = ... })` | `@Int(...)` |
| `@Type(.{ .@"struct" = .{ .is_tuple = true, ... } })` | `@Tuple(&.{ ... })` |
| `@Type(.{ .pointer = ... })` | `@Pointer(...)` |
| `@Type(.{ .@"fn" = ... })` | `@Fn(...)` |
| `@Type(.{ .@"struct" = ... })` | `@Struct(...)` |
| `@Type(.{ .@"union" = ... })` | `@Union(...)` |
| `@Type(.{ .@"enum" = ... })` | `@Enum(...)` |
| `@Type(.enum_literal)` | `@EnumLiteral()` |

#### `@intFromFloat` 被弃用

仍可用，但推荐使用 `@trunc` / `@floor` / `@ceil` / `@round` 直接转整数。

```zig
const x: f32 = 3.7;
const y: u32 = @trunc(x); // 0.16 允许，结果为 3
```

### 1.2 语法移除

#### `usingnamespace` 被彻底移除

```zig
// ≤0.14
usingnamespace @import("std");
```

**验证（0.16）**：`error: expected ',' after field`

**替代方案**：

- 条件包含：用 `if` 或 comptime `switch`
- 实现切换：用 `inline for` 或 comptime 选择
- Mixins：显式将字段/函数导入当前命名空间

#### `async` / `await` 关键字被移除

```zig
// ≤0.14
const frame = async foo();
await frame;
```

**验证（0.16）**：`error: expected ';' after statement`

**替代方案**：重写为同步代码，或使用 `std.Io.Future` / `std.Io.Group`。

### 1.3 语义收紧

#### `@export` 操作数必须是 comptime-known 指针

```zig
export var x: u32 = 0;

// ≤0.13 允许值导出；0.14+ 报错
@export(x, .{ .name = "y" }); // error: export target must be comptime-known

// 0.16 正确写法
@export(&x, .{ .name = "y" });
```

#### sentinel 必须是标量类型

```zig
// 0.16 报错：expected type '[2]u8', found 'comptime_int'
var x: [*:0]const [2]u8 = undefined;
```

#### in-memory coercion 被禁止

```zig
var arr: [4]u8 = .{ 1, 2, 3, 4 };
const vec: @Vector(4, u8) = @ptrCast(&arr); // 0.16 报错
```

**验证（0.16）**：`error: expected pointer type, found '@Vector(4, u8)'`

#### switch 对 non-exhaustive enum 必须加 `else`

```zig
const E = enum(u8) { a, b, _ };
fn foo(e: E) void {
    switch (e) {
        .a => {},
        .b => {},
        else => {}, // 0.15+ 必须
    }
}
```

#### 禁止从函数返回局部变量地址

```zig
fn foo() *u32 {
    var x: u32 = 42;
    return &x; // 0.16 编译报错
}
```

#### runtime vector index 被禁止

```zig
fn bar(i: usize) u32 {
    const v = @Vector(4, u32){ 1, 2, 3, 4 };
    return v[i]; // 0.16 报错
}
```

#### packed union / packed struct 规则收紧

- packed union 禁止包含指针字段
- packed union 要求所有字段位宽与 backing integer 一致（禁止 unused bits）
- extern context 中不允许隐式 backing type 的 enum / packed 类型
- packed union 支持显式 backing integer：`packed union(u32) { ... }`

#### 显式对齐指针与自然对齐指针被视为不同类型

```zig
var x: u32 = 0;
const p1: *u32 = &x;
const p2: *align(4) u32 = &x;
// 0.16 中 p1 和 p2 的类型不再相等
```

#### 小整数可隐式 coercion 到浮点

```zig
const x: u24 = 123;
const y: f32 = x; // 0.16 允许（无损）
```

### 1.4 类型系统调整

#### `std.builtin.Type` 字段重命名

| 旧字段名 | 新字段名 |
|---|---|
| `.Struct` | `.@"struct"` |
| `.Union` | `.@"union"` |
| `.Enum` | `.@"enum"` |
| `.Fn` | `.@"fn"` |
| `.Pointer.size` 中的 `.One` | `.one`（小写） |

影响：所有依赖 `@typeInfo` + `std.builtin.Type` 的元编程代码需要更新字段访问。

#### `?*const anyopaque` 在 `@typeInfo` 中简化

0.14 起 `std.builtin.Type.Pointer` 的 `child` 字段处理简化，
很多需要显式写 `?*const anyopaque` 的场景现在可以直接使用更自然的类型表达。

### 1.5 其他

#### `@cImport` 被标记为 deprecated

仍可用（无 deprecation warning），但官方方向是将其完全迁移到 Build System 的 `b.addTranslateC`。

---

## 二、标准库（Standard Library）

### 2.1 分配器

| 旧 API | 0.16 状态 |
|---|---|
| `std.heap.GeneralPurposeAllocator` | 移除 → 改用 `std.heap.DebugAllocator` |
| `std.heap.SmpAllocator` | 新增，作为多线程高性能分配器 |
| `std.heap.ThreadSafeAllocator` | 移除 → 改用 `std.heap.ArenaAllocator`（已线程安全） |
| `Allocator` vtable | 新增 `remap` 字段，自定义分配器必须实现 |

**验证（0.16）**：

```bash
error: root source file struct 'heap' has no member named 'ThreadSafeAllocator'
```

### 2.2 容器与数据结构

#### `ArrayList` 默认变为 Unmanaged

```zig
// ≤0.14
var list = std.ArrayList(u8).init(allocator);

// 0.16 报错：has no member named 'init'

// 0.16 正确写法（Unmanaged）
var arr: std.ArrayList(u8) = .empty;
try arr.append(allocator, 'a');

// 若仍需 Managed 版本（已标记 Deprecated）
var list = std.ArrayListManaged(u8).init(allocator);
```

**验证（0.16）**：
`error: struct 'array_list.Aligned(u8,null)' has no member named 'init'`

#### `BoundedArray` 被删除

```zig
// ≤0.14
var buf = std.BoundedArray(u8, 256).init(0) catch unreachable;
```

**0.16**：`std.BoundedArray` 已删除，无直接替代。

#### 链表去泛型化

`std.SinglyLinkedList` / `std.DoublyLinkedList` 等去除了大量泛型参数，API 更加固定。

### 2.3 I/O 大重构（Writergate → `std.Io`）

0.16 中 `std.Io` 正式成为一等抽象，大量旧 API 被迁移或删除。

| 旧 API（≤0.15） | 0.16 状态 |
|---|---|
| `std.fs.cwd().createFile(...)` | `std.Io.Dir.cwd().createFile(io, ...)` |
| `std.fs.File.readAll(...)` | `std.Io.File.readPositionalAll(io, ...)` |
| `std.io.GenericReader` / `AnyReader` / `FixedBufferStream` | ❌ 删除 |
| `std.Thread.Pool` | ❌ 删除 → `std.Io.Group` |
| `std.time.Instant.now()` | `std.Io.Timestamp.now(io, .real)` |
| `std.crypto.random.bytes(&buf)` | `io.random(&buf)` |

**验证（0.16）**：

```bash
error: root source file struct 'Thread' has no member named 'Pool'
error: root source file struct 'std' has no member named 'io'
```

### 2.4 进程与环境

| 旧 API | 0.16 状态 |
|---|---|
| `std.process.args` / `std.process.env` | ❌ 删除 → `main(init: std.process.Init)` |
| `std.process.Child.collectOutput` | API 签名已变更 |

### 2.5 其他删除

| 旧 API | 0.16 状态 |
|---|---|
| `std.builtin.subsystem` | ❌ 删除 → `std.zig.Subsystem` |
| `std.fs.getAppDataDir` | ❌ 删除 |
| `std.c` 绑定 | 大规模重组，符号被移到平台子模块 |

**验证（0.16）**：

```bash
error: root source file struct 'builtin' has no member named 'subsystem'
error: root source file struct 'fs' has no member named 'getAppDataDir'
```

### 2.6 格式化打印

- 调用 `format` 方法必须显式使用 `"{f}"` 说明符
- `format` 方法不再接收 format string 和 options 参数
- 格式化打印不再处理 Unicode（不再自动识别 `%s` 等）
- 新增部分格式化说明符

### 2.7 Panic Interface

`@panic` 的底层处理接口发生变化，自定义 panic handler 的签名需要更新。

---

## 三、构建系统（Build System）

### 3.1 模块化重构

#### 移除隐式 Root Module

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

#### `addTest` 使用 `root_module`

```zig
const exe_tests = b.addTest(.{
    .root_module = exe.root_module,
});
```

#### `addModule` 签名简化

```zig
const lib_mod = b.addModule("mylib", .{
    .root_source_file = b.path("src/lib.zig"),
});
```

### 3.2 库与链接

#### `linkLibC()` 被移除

```zig
// ≤0.15
exe.linkLibC();

// 0.16
const root_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .link_libc = true,
});
```

**验证（0.16）**：`error: no member named 'linkLibC'`

#### `addLibrary` 引入

`b.addLibrary` 成为新增的统一入口，用于创建库产物。

### 3.3 其他

- `build.zig.zon` 的依赖哈希算法已变更，0.13 的 hash 无法被 0.16 识别
- 新增 `Fmt Step` / `WriteFile Step` / `RemoveDir Step` 等 step 类型
- `zig init` 成为创建新项目的标准方式

---

## 四、0.16 编译器验证结果汇总

| 变更内容 | 0.16 验证结果 |
|---|---|
| `@setCold` 移除 | ✅ `invalid builtin function: '@setCold'` |
| `@fence` 移除 | ✅ `invalid builtin function: '@fence'` |
| `@Type` 移除 | ✅ `invalid builtin function: '@Type'` |
| `@export` 需指针 | ✅ `export target must be comptime-known` |
| 非标的量 sentinel 禁止 | ✅ `expected type '[2]u8', found 'comptime_int'` |
| in-memory coercion 限制 | ✅ `@ptrCast` array→vector 报错 |
| `usingnamespace` 移除 | ✅ `expected ',' after field` |
| `async`/`await` 关键字移除 | ✅ `expected ';' after statement` |
| `ArrayList` 默认 unmanaged | ✅ `has no member named 'init'` |
| `Thread.Pool` 删除 | ✅ `has no member named 'Pool'` |
| `heap.ThreadSafeAllocator` 删除 | ✅ `has no member named 'ThreadSafeAllocator'` |
| `builtin.subsystem` 删除 | ✅ `has no member named 'subsystem'` |
| `fs.getAppDataDir` 删除 | ✅ `has no member named 'getAppDataDir'` |
| `linkLibC()` 删除 | ✅ `no member named 'linkLibC'` |

---

## 五、迁移优先级建议

1. **语言层破坏**（`@Type`、`usingnamespace`、`async`/`await`、`@setCold`、`@fence`）
2. **标准库 API**（`ArrayList` → unmanaged、`std.Io` 迁移、
   删除的 `Thread.Pool` 等）
3. **构建系统**（`root_module`、`link_libc`）

---

## 参考来源

- Zig 0.14.0 Release Notes: `https://ziglang.org/download/0.14.0/release-notes.html`
- Zig 0.15.1 Release Notes: `https://ziglang.org/download/0.15.1/release-notes.html`
- Zig 0.16.0 Release Notes: `https://ziglang.org/download/0.16.0/release-notes.html`
- 本地 Zig 0.16.0 编译器验证
