# build.zig 迁移要点

## API 签名变化

### `addExecutable` 不再接受 `root_source_file`

0.15 及以前：

```zig
const exe = b.addExecutable(.{
    .name = "myapp",
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
```

0.16（已实验验证）：

```zig
const root_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});

const exe = b.addExecutable(.{
    .name = "myapp",
    .root_module = root_mod,
});
```

### `addModule` 签名变化

0.15 及以前：先 `createModule` 再传给 `addModule(name, module)`。

0.16（已实验验证）：`addModule` 直接接受 `Module.CreateOptions`：

```zig
const lib_mod = b.addModule("mylib", .{
    .root_source_file = b.path("src/lib.zig"),
    .target = target,
    .optimize = optimize,
});

const exe_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});

exe_mod.addImport("mylib", lib_mod);
```

### `linkLibC()` 已移除

0.15：

```zig
exe.linkLibC();
```

0.16（已实验验证）：

```zig
const root_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
    .link_libc = true,
});
```

### `addTest` 使用 `root_module`

0.15：

```zig
const exe_tests = b.addTest(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
```

0.16（已实验验证）：

```zig
const exe_tests = b.addTest(.{
    .root_module = exe.root_module,
});
```

## 模块依赖与导入

模块间的导入关系通过 `addImport` 建立：

```zig
lib_mod.addImport("utils", utils_mod);
exe_mod.addImport("mylib", lib_mod);
```

源码中使用：

```zig
const mylib = @import("mylib");
```

## 链接系统库

0.15：

```zig
exe.linkSystemLibrary("zstd");
```

0.16：

```zig
exe.root_module.linkSystemLibrary("zstd", .{});
```

## C Translation（无需 `zig fetch`）

```zig
const translate_c = b.addTranslateC(.{
    .root_source_file = b.path("include/mylib.h"),
    .target = target,
    .optimize = optimize,
});

exe.root_module.addImport("c", translate_c.createModule());
```

## 新增 CLI 参数

- `zig build --fork=[path]` — 本地覆盖依赖包
- `zig build --fetch[=mode]` — 将依赖拉取到项目本地目录
- `zig build --test-timeout <timeout>` — 单元测试超时（需带单位：ns/us/ms/s/m/h）
- `zig build --error-style [verbose|minimal]` — 控制错误输出风格
- `zig build --multiline-errors [newline|...]` — 控制多行错误消息格式
