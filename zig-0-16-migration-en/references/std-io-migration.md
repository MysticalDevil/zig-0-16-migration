# std.Io Migration Examples

## Core Change

Zig 0.16.0 introduces `std.Io` as the unified interface for I/O, concurrency, cancellation, and synchronization. Any operation that may block control flow or introduce nondeterminism requires an `Io` instance to be passed in.

## `main` Signature Change

0.15 and earlier:

```zig
pub fn main() !void { ... }
```

0.16 (Juicy Main):

```zig
pub fn main(init: std.process.Init) !void {
    _ = init;
    ...
}
```

## Creating an `Io` Instance

### Single-threaded scenario

```zig
var threaded: std.Io.Threaded = .init_single_threaded;
const io = threaded.io();
```

### Multi-threaded scenario

```zig
var threaded = std.Io.Threaded.init(allocator, .{});
const io = threaded.io();
```

## File I/O Migration

### 0.15 old style

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

### 0.16 new style (experimentally verified)

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    const dir = std.Io.Dir.cwd();

    // Write file
    {
        const file = try dir.createFile(io, "test.txt", .{ .truncate = true });
        defer file.close(io);
        try file.writeStreamingAll(io, "hello from std.Io\n");
    }

    // Read file
    {
        const file = try dir.openFile(io, "test.txt", .{});
        defer file.close(io);

        var buf: [256]u8 = undefined;
        const n = try file.readPositionalAll(io, &buf, 0);
        std.debug.print("read {d} bytes: {s}", .{ n, buf[0..n] });
    }
}
```

## Common API Mapping

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

## Process API Migration

### 0.15 old style

```zig
var child = std.process.Child.init(argv, gpa);
child.stdin_behavior = .Pipe;
child.stdout_behavior = .Pipe;
try child.spawn();
```

### 0.16 new style

```zig
const result = try std.process.spawn(io, .{
    .argv = argv,
    .stdin = .pipe,
    .stdout = .pipe,
});
```

### Run child process and capture output

```zig
const result = try std.process.run(allocator, io, .{
    .argv = argv,
});
```

## Sync Primitive Migration

| 0.15 API | 0.16 API |
|---|---|
| `std.Thread.ResetEvent` | `std.Io.Event` |
| `std.Thread.WaitGroup` | `std.Io.Group` |
| `std.Thread.Futex` | `std.Io.Futex` |
| `std.Thread.Mutex` | `std.Io.Mutex` |
| `std.Thread.Condition` | `std.Io.Condition` |
| `std.Thread.Semaphore` | `std.Io.Semaphore` |
| `std.Thread.RwLock` | `std.Io.RwLock` |

## Time and Randomness Migration

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

## Using `Io` in Tests

```zig
const io = std.testing.io;
// Similar usage to std.testing.allocator
```
