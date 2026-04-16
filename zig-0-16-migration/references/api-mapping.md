# 0.15 -> 0.16 常用 API 对照表

## 标准库删除/重命名

| 0.15 API | 0.16 状态 | 替代方案 |
|---|---|---|
| `std.Thread.Pool` | ❌ 已删除 | `std.Io.Group` 或 `std.Io.Future` |
| `std.heap.ThreadSafeAllocator` | ❌ 已删除 | `std.heap.ArenaAllocator`（现已无锁线程安全） |
| `std.io.GenericReader` | ❌ 已删除 | `std.Io.Reader` |
| `std.io.AnyReader` | ❌ 已删除 | `std.Io.Reader` |
| `std.io.FixedBufferStream` | ❌ 已删除 | `std.Io.Reader` / `std.Io.Writer` |
| `std.fs.getAppDataDir` | ❌ 已删除 | 无直接替代，需自行实现平台相关逻辑 |
| `std.fs.Dir` | ⚠️ 迁移 | `std.Io.Dir` |
| `std.fs.File` | ⚠️ 迁移 | `std.Io.File` |
| `std.fs.path` | ⚠️ 迁移 | `std.Io.Dir.path` |
| `std.fs.max_path_bytes` | ⚠️ 迁移 | `std.Io.Dir.max_path_bytes` |
| `std.fs.max_name_bytes` | ⚠️ 迁移 | `std.Io.Dir.max_name_bytes` |
| `std.fs.cwd()` | ⚠️ 迁移 | `std.Io.Dir.cwd()` |
| `std.process.getCwd` | ⚠️ 重命名 | `std.process.currentPath(io, buf)` |
| `std.process.getCwdAlloc` | ⚠️ 重命名 | `std.process.currentPathAlloc(io, allocator)` |
| `std.process.Child.init` | ⚠️ 迁移 | `std.process.spawn(io, .{ .argv = argv })` |
| `std.process.Child.run` | ⚠️ 迁移 | `std.process.run(allocator, io, .{ ... })` |
| `std.process.execv` | ⚠️ 迁移 | `std.process.replace(io, .{ .argv = argv })` |
| `std.process.args` | ❌ 已删除 | 通过 `std.process.Init` 在 `main(init)` 中获取 |
| `std.process.env` | ❌ 已删除 | 通过 `std.process.Init` 在 `main(init)` 中获取 |
| `std.builtin.subsystem` | ❌ 已删除 | `std.Target.SubSystem` 或 `std.process.Init` 相关字段 |
| `std.Thread.ResetEvent` | ⚠️ 迁移 | `std.Io.Event` |
| `std.Thread.WaitGroup` | ⚠️ 迁移 | `std.Io.Group` |
| `std.Thread.Futex` | ⚠️ 迁移 | `std.Io.futexWait` / `std.Io.futexWake`（无直接类型替代） |
| `std.Thread.Mutex` | ⚠️ 迁移 | `std.Io.Mutex` |
| `std.Thread.Condition` | ⚠️ 迁移 | `std.Io.Condition` |
| `std.Thread.Semaphore` | ⚠️ 迁移 | `std.Io.Semaphore` |
| `std.Thread.RwLock` | ⚠️ 迁移 | `std.Io.RwLock` |
| `std.mem.indexOf` | ⚠️ 重命名 | `std.mem.find` |
| `std.mem.indexOfScalar` 等 | ⚠️ 重命名 | `std.mem.findScalar` 等对应函数 |
| `std.mem.cut` | ✅ 新增 | 新函数，用于分割切片 |
| `std.crypto.random.bytes` | ⚠️ 迁移 | `io.random(&buffer)` |
| `std.time.Instant` | ⚠️ 迁移 | `std.Io.Timestamp` |
| `std.time.Timer` | ⚠️ 迁移 | `std.Io.Timestamp` |
| `std.time.timestamp` | ⚠️ 迁移 | `std.Io.Timestamp.now(io, .real)` |

## 文件系统 API 详细对照

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
| `fs.Dir.rename` | `std.Io.Dir.rename`（签名改变，接受两个 Dir + Io） |
| `fs.File.Mode` | `std.Io.File.Permissions` |
| `fs.File.default_mode` | `std.Io.File.Permissions.default_file` |
| `fs.File.getEndPos` | `std.Io.File.length(io)` |
| `fs.File.seekTo` | `std.Io.File.Reader.seekTo` |
| `fs.File.read` | `std.Io.File.readStreaming` |
| `fs.File.write` | `std.Io.File.writeStreaming` |
| `fs.File.readAll` | `std.Io.File.readPositionalAll` |
| `fs.File.writeAll` | `std.Io.File.writeStreamingAll` |

## Build System API 对照

| 0.15 API | 0.16 API |
|---|---|
| `b.addExecutable(.{ .root_source_file = ... })` | `b.createModule(...)` + `b.addExecutable(.{ .root_module = mod })` |
| `b.addModule(name, module_ptr)` | `b.addModule(name, .{ .root_source_file = ... })` |
| `exe.linkLibC()` | `b.createModule(.{ ..., .link_libc = true })` |
| `exe.linkSystemLibrary("xxx")` | `exe.root_module.linkSystemLibrary("xxx", .{})` |
| `b.addTest(.{ .root_source_file = ... })` | `b.addTest(.{ .root_module = mod })` |
| `@cImport({ ... })` | `b.addTranslateC(...)` + `exe.root_module.addImport("c", translate_c.createModule())` |

## 语言 builtin 对照

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
| `@intFromFloat` | `@trunc`（`@intFromFloat` 已弃用） |

## 快速迁移检查清单

- [ ] 搜索 `std.Thread.Pool` → 替换为 `std.Io.Group` / `std.Io.Future`
- [ ] 搜索 `heap.ThreadSafeAllocator` → 评估是否可直接用 `ArenaAllocator`
- [ ] 搜索 `std.io.GenericReader` / `AnyReader` / `FixedBufferStream` → 迁移到 `std.Io`
- [ ] 搜索 `fs.getAppDataDir` → 手写平台逻辑或移除
- [ ] 搜索 `fs.` 文件/路径操作 → 检查是否已迁移到 `std.Io.Dir`
- [ ] 搜索 `process.args` / `process.env` → 改为 `main(init: std.process.Init)`
- [ ] 搜索 `builtin.subsystem` → 查找新的位置
- [ ] 搜索 `@Type` → 按对照表替换为独立 builtins
- [ ] 搜索 `@cImport` → 改为 `build.zig` 中 `addTranslateC`
- [ ] 搜索 `linkLibC()` → 改为 `createModule` 中设置 `link_libc = true`
