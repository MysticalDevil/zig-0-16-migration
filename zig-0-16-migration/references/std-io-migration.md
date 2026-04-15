# std.Io 迁移样例

## 核心变化

Zig 0.16.0 引入了 `std.Io` 作为统一的 I/O、并发、取消、同步接口。所有可能阻塞控制流或引入非确定性的操作都需要传入 `Io` 实例。

## main 函数签名变化

0.15 及以前：

```zig
pub fn main() !void { ... }
```

0.16（Juicy Main）：

```zig
pub fn main(init: std.process.Init) !void {
    _ = init;
    ...
}
```

## 创建 Io 实例

### 单线程场景

```zig
var threaded: std.Io.Threaded = .init_single_threaded;
const io = threaded.io();
```

### 多线程场景

```zig
var threaded = std.Io.Threaded.init(allocator, .{});
const io = threaded.io();
```

## 文件 I/O 迁移

### 0.15 旧写法

```zig
const std = @import("std");

pub fn main() !void {
    const file = try std.fs.cwd().createFile("test.txt", .{ .truncate = true });
    defer file.close();
    try file.writeAll("hello\n");

    const file2 = try std.fs.cwd().openFile("test.txt", .{});
    defer file2.close();
    var buf: [256]u8 = undefined;
    const n = try file2.readAll(&buf);
}
```

### 0.16 新写法（已实验验证）

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    const dir = std.Io.Dir.cwd();

    // 写文件
    {
        const file = try dir.createFile(io, "test.txt", .{ .truncate = true });
        defer file.close(io);
        try file.writeStreamingAll(io, "hello from std.Io\n");
    }

    // 读文件
    {
        const file = try dir.openFile(io, "test.txt", .{});
        defer file.close(io);

        var buf: [256]u8 = undefined;
        const n = try file.readPositionalAll(io, &buf, 0);
        std.debug.print("read {d} bytes: {s}", .{ n, buf[0..n] });
    }
}
```

## 常用 API 对照

| 0.15 API | 0.16 API |
|---|---|
| `std.fs.cwd()` | `std.Io.Dir.cwd()` |
| `dir.createFile(path, flags)` | `dir.createFile(io, path, flags)` |
| `dir.openFile(path, flags)` | `dir.openFile(io, path, flags)` |
| `file.close()` | `file.close(io)` |
| `file.read(buffer)` | `file.readStreaming(io, &.{buffer})` |
| `file.write(buffer)` | `file.writeStreamingAll(io, buffer)` |
| `file.readAll(buffer)` | `file.readPositionalAll(io, buffer, 0)` |
| `file.writeAll(buffer)` | `file.writeStreamingAll(io, buffer)` |
| `file.getEndPos()` | `file.length(io)` |
| `file.seekTo(pos)` | `file.reader(io, buf).seekTo(pos)` |
| `file.reader()` | `file.reader(io, buffer)` |
| `file.writer()` | `file.writer(io, buffer)` |
| `std.fs.path` | `std.Io.Dir.path` |
| `std.process.getCwd` | `std.process.currentPath(io, buffer)` |

## 进程 API 迁移

### 0.15 旧写法

```zig
var child = std.process.Child.init(argv, gpa);
child.stdin_behavior = .Pipe;
child.stdout_behavior = .Pipe;
try child.spawn();
```

### 0.16 新写法

```zig
const result = try std.process.spawn(io, .{
    .argv = argv,
    .stdin = .pipe,
    .stdout = .pipe,
});
```

### 运行子进程并捕获输出

```zig
const result = try std.process.run(allocator, io, .{
    .argv = argv,
});
```

## 同步原语迁移

| 0.15 API | 0.16 API |
|---|---|
| `std.Thread.ResetEvent` | `std.Io.Event` |
| `std.Thread.WaitGroup` | `std.Io.Group` |
| `std.Thread.Futex` | `std.Io.Futex` |
| `std.Thread.Mutex` | `std.Io.Mutex` |
| `std.Thread.Condition` | `std.Io.Condition` |
| `std.Thread.Semaphore` | `std.Io.Semaphore` |
| `std.Thread.RwLock` | `std.Io.RwLock` |

## 时间和随机数迁移

### 0.15 style — randomness

```zig
std.crypto.random.bytes(&buffer);
```

### 0.16 style — randomness

```zig
io.random(&buffer);
io.randomSecure(&buffer);
```

### 0.15 style — time

```zig
std.time.Instant.now()
std.time.Timer.start()
```

### 0.16 style — time

```zig
std.Io.Timestamp.now(io)
std.Io.Clock.resolution(io)
```

## 测试中的 Io

```zig
const io = std.testing.io;
// 类似 std.testing.allocator 的用法
```
