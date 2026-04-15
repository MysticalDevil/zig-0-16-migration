# 0.15 -> 0.16 Common API Mapping

## Standard Library Removals / Renames

| 0.15 API | 0.16 Status | Replacement |
|---|---|---|
| `std.Thread.Pool` | ❌ Removed | `std.Io.Group` or `std.Io.Future` |
| `std.heap.ThreadSafeAllocator` | ❌ Removed | `std.heap.ArenaAllocator` (now lock-free and thread-safe) |
| `std.io.GenericReader` | ❌ Removed | `std.Io.Reader` |
| `std.io.AnyReader` | ❌ Removed | `std.Io.Reader` |
| `std.io.FixedBufferStream` | ❌ Removed | `std.Io.Reader` / `std.Io.Writer` |
| `std.fs.getAppDataDir` | ❌ Removed | No direct replacement; implement platform-specific logic manually |
| `std.fs.Dir` | ⚠️ Migrated | `std.Io.Dir` |
| `std.fs.File` | ⚠️ Migrated | `std.Io.File` |
| `std.fs.path` | ⚠️ Migrated | `std.Io.Dir.path` |
| `std.fs.max_path_bytes` | ⚠️ Migrated | `std.Io.Dir.max_path_bytes` |
| `std.fs.max_name_bytes` | ⚠️ Migrated | `std.Io.Dir.max_name_bytes` |
| `std.fs.cwd()` | ⚠️ Migrated | `std.Io.Dir.cwd()` |
| `std.process.getCwd` | ⚠️ Renamed | `std.process.currentPath(io, buf)` |
| `std.process.getCwdAlloc` | ⚠️ Renamed | `std.process.currentPathAlloc(io, allocator)` |
| `std.process.Child.init` | ⚠️ Migrated | `std.process.spawn(io, .{ .argv = argv })` |
| `std.process.Child.run` | ⚠️ Migrated | `std.process.run(allocator, io, .{ ... })` |
| `std.process.execv` | ⚠️ Migrated | `std.process.replace(io, .{ .argv = argv })` |
| `std.process.args` | ❌ Removed | Obtain via `std.process.Init` in `main(init)` |
| `std.process.env` | ❌ Removed | Obtain via `std.process.Init` in `main(init)` |
| `std.builtin.subsystem` | ❌ Removed | `std.Target.SubSystem` or fields in `std.process.Init` |
| `std.Thread.ResetEvent` | ⚠️ Migrated | `std.Io.Event` |
| `std.Thread.WaitGroup` | ⚠️ Migrated | `std.Io.Group` |
| `std.Thread.Futex` | ⚠️ Migrated | `std.Io.Futex` |
| `std.Thread.Mutex` | ⚠️ Migrated | `std.Io.Mutex` |
| `std.Thread.Condition` | ⚠️ Migrated | `std.Io.Condition` |
| `std.Thread.Semaphore` | ⚠️ Migrated | `std.Io.Semaphore` |
| `std.Thread.RwLock` | ⚠️ Migrated | `std.Io.RwLock` |
| `std.mem.indexOf` | ⚠️ Renamed | `std.mem.find` |
| `std.mem.indexOfScalar` etc. | ⚠️ Renamed | `std.mem.findScalar` etc. |
| `std.mem.cut` | ✅ New | New function for splitting slices |
| `std.crypto.random.bytes` | ⚠️ Migrated | `io.random(&buffer)` |
| `std.time.Instant` | ⚠️ Migrated | `std.Io.Timestamp` |
| `std.time.Timer` | ⚠️ Migrated | `std.Io.Timestamp` |
| `std.time.timestamp` | ⚠️ Migrated | `std.Io.Timestamp.now(io)` |

## File System API Mapping

| 0.15 API | 0.16 API |
|---|---|
| `fs.copyFileAbsolute` | `std.Io.Dir.copyFileAbsolute` |
| `fs.makeDirAbsolute` | `std.Io.Dir.createDirAbsolute` |
| `fs.deleteDirAbsolute` | `std.Io.Dir.deleteDirAbsolute` |
| `fs.openDirAbsolute` | `std.Io.Dir.openDirAbsolute` |
| `fs.openFileAbsolute` | `std.Io.Dir.openFileAbsolute` |
| `fs.accessAbsolute` | `std.Io.Dir.accessAbsolute` |
| `fs.createFileAbsolute` | `std.Io.Dir.createFileAbsolute` |
| `fs.deleteFileAbsolute` | `std.Io.Dir.deleteFileAbsolute` |
| `fs.renameAbsolute` | `std.Io.Dir.renameAbsolute` |
| `fs.readLinkAbsolute` | `std.Io.Dir.readLinkAbsolute` |
| `fs.symLinkAbsolute` | `std.Io.Dir.symLinkAbsolute` |
| `fs.realpath` | `std.Io.Dir.realPathFileAbsolute` |
| `fs.rename` | `std.Io.Dir.rename` |
| `fs.Dir.makeDir` | `std.Io.Dir.createDir` |
| `fs.Dir.makePath` | `std.Io.Dir.createDirPath` |
| `fs.Dir.makeOpenDir` | `std.Io.Dir.createDirPathOpen` |
| `fs.Dir.rename` | `std.Io.Dir.rename` (signature changed: takes two `Dir` + `Io`) |
| `fs.File.Mode` | `std.Io.File.Permissions` |
| `fs.File.default_mode` | `std.Io.File.Permissions.default_file` |
| `fs.File.getEndPos` | `std.Io.File.length(io)` |
| `fs.File.seekTo` | `std.Io.File.Reader.seekTo` |
| `fs.File.read` | `std.Io.File.readStreaming` |
| `fs.File.write` | `std.Io.File.writeStreaming` |
| `fs.File.readAll` | `std.Io.File.readPositionalAll` |
| `fs.File.writeAll` | `std.Io.File.writeStreamingAll` |

## Build System API Mapping

| 0.15 API | 0.16 API |
|---|---|
| `b.addExecutable(.{ .root_source_file = ... })` | `b.createModule(...)` + `b.addExecutable(.{ .root_module = mod })` |
| `b.addModule(name, module_ptr)` | `b.addModule(name, .{ .root_source_file = ... })` |
| `exe.linkLibC()` | `b.createModule(.{ ..., .link_libc = true })` |
| `exe.linkSystemLibrary("xxx")` | `exe.root_module.linkSystemLibrary("xxx", .{})` |
| `b.addTest(.{ .root_source_file = ... })` | `b.addTest(.{ .root_module = mod })` |
| `@cImport({ ... })` | `b.addTranslateC(...)` + `exe.root_module.addImport("c", translate_c.createModule())` |

## Language Builtin Mapping

| 0.15 | 0.16 |
|---|---|
| `@Type(.{ .int = ... })` | `@Int(.unsigned, 10)` |
| `@Type(.{ .@"struct" = .{ .is_tuple = true, ... } })` | `@Tuple(&.{ u32, f64 })` |
| `@Type(.{ .pointer = ... })` | `@Pointer(.one, .{ .@"const" = true }, u8, null)` |
| `@Type(.{ .@"fn" = ... })` | `@Fn(&.{ u32, f64 }, &@splat(.{}), void, .{})` |
| `@Type(.{ .@"struct" = ... })` | `@Struct(.auto, null, &.{"x"}, &.{u32}, &@splat(.{}))` |
| `@Type(.{ .@"union" = ... })` | `@Union(.auto, null, &.{"x"}, &.{u32}, &@splat(.{}))` |
| `@Type(.{ .@"enum" = ... })` | `@Enum(u8, .exhaustive, &.{"a"}, &.{0})` |
| `@Type(.enum_literal)` | `@EnumLiteral()` |
| `@intFromFloat` | `@trunc` (`@intFromFloat` is deprecated) |

## Quick Migration Checklist

- [ ] Search `std.Thread.Pool` → replace with `std.Io.Group` / `std.Io.Future`
- [ ] Search `heap.ThreadSafeAllocator` → evaluate if `ArenaAllocator` is sufficient
- [ ] Search `std.io.GenericReader` / `AnyReader` / `FixedBufferStream` → migrate to `std.Io`
- [ ] Search `fs.getAppDataDir` → hand-roll platform logic or remove
- [ ] Search `fs.` file/path operations → check if migrated to `std.Io.Dir`
- [ ] Search `process.args` / `process.env` → use `main(init: std.process.Init)`
- [ ] Search `builtin.subsystem` → find new location
- [ ] Search `@Type` → replace with independent builtins per mapping table
- [ ] Search `@cImport` → use `addTranslateC` in `build.zig`
- [ ] Search `linkLibC()` → set `link_libc = true` in `createModule`
