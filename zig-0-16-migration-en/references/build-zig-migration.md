# build.zig Migration Highlights

## API Signature Changes

### `addExecutable` no longer accepts `root_source_file`

0.15 and earlier:

```zig
const exe = b.addExecutable(.{
    .name = "myapp",
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
```

0.16 (experimentally verified):

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

### `addModule` signature change

0.15 and earlier: first `createModule`, then pass it to `addModule(name, module)`.

0.16 (experimentally verified): `addModule` now directly accepts `Module.CreateOptions`:

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

### `linkLibC()` removed

0.15:

```zig
exe.linkLibC();
```

0.16 (experimentally verified):

```zig
const root_mod = b.createModule(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
    .link_libc = true,
});
```

### `addTest` uses `root_module`

0.15:

```zig
const exe_tests = b.addTest(.{
    .root_source_file = b.path("src/main.zig"),
    .target = target,
    .optimize = optimize,
});
```

0.16 (experimentally verified):

```zig
const exe_tests = b.addTest(.{
    .root_module = exe.root_module,
});
```

## Module Dependencies and Imports

Inter-module imports are established via `addImport`:

```zig
lib_mod.addImport("utils", utils_mod);
exe_mod.addImport("mylib", lib_mod);
```

In source code:

```zig
const mylib = @import("mylib");
```

## Linking System Libraries

0.15:

```zig
exe.linkSystemLibrary("zstd");
```

0.16:

```zig
exe.root_module.linkSystemLibrary("zstd", .{});
```

## C Translation (no `zig fetch` required)

```zig
const translate_c = b.addTranslateC(.{
    .root_source_file = b.path("include/mylib.h"),
    .target = target,
    .optimize = optimize,
});

exe.root_module.addImport("c", translate_c.createModule());
```

## New CLI Flags

- `zig build --fork=[path]` — Override dependencies locally
- `zig build --fetch[=mode]` — Fetch dependencies into a project-local directory
- `zig build --test-timeout <timeout>` — Unit test timeout (requires unit: ns/us/ms/s/m/h)
- `zig build --error-style [verbose|minimal]` — Control error output style
- `zig build --multiline-errors [newline|...]` — Control multi-line error message formatting
