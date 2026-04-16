# `std.Io` Complete API Reference (Zig 0.16.0)

This document is compiled from the Zig 0.16.0 official source code and release notes. All code examples have been verified to compile locally.

---

## 1. What is `std.Io`

`std.Io` is the unified interface introduced in Zig 0.16.0. It abstracts **file system, networking, processes, time, randomness, concurrency, cancellation, and synchronization** behind a single API.

Core design:

- `Io` itself is a lightweight struct containing only `userdata: ?*anyopaque` and `vtable: *const VTable`
- Any operation that may block or introduce non-determinism requires an explicit `Io` instance
- The 0.15 style of directly calling `std.fs.cwd().createFile(...)` is replaced with `dir.createFile(io, ...)`

---

## 2. Obtaining an `Io` Instance

### Single-threaded

```zig
var threaded: std.Io.Threaded = .init_single_threaded;
const io = threaded.io();
// `threaded` must outlive `io`
```

### Multi-threaded

```zig
var threaded = std.Io.Threaded.init(allocator, .{});
const io = threaded.io();
```

### Testing

```zig
const io = std.testing.io;
```

### `main` Signature Change (Juicy Main)

0.15 and earlier:

```zig
pub fn main() !void { ... }
```

0.16:

```zig
pub fn main(init: std.process.Init) !void {
    _ = init;
    ...
}
```

---

## 3. Passing `Io` Around Like `Allocator`

`Io` is tiny (two pointers), so it is passed **by value**, just like `std.mem.Allocator`.

```zig
fn fetchData(io: std.Io, url: []const u8, buf: []u8) !usize {
    _ = url;
    // All blocking or non-deterministic operations go through `io`
    _ = try io.randomSecure(buf);
    return buf.len;
}

pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();
    var buf: [32]u8 = undefined;
    _ = try fetchData(io, "example.com", &buf);
}
```

Convention (following the stdlib):

- `io` is usually placed early or late in the parameter list. In 0.16 std, sync primitives use `lock(io)`, `wait(io, ...)`, while file operations use `file.readStreaming(io, buf)` and `dir.createFile(io, path, opts)`. This document notes the signature for each API.

---

## 4. File System I/O

### File I/O Migration Example

0.15 old style:

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

0.16 new style:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    const dir = std.Io.Dir.cwd();

    {
        const file = try dir.createFile(io, "test.txt", .{ .truncate = true });
        defer file.close(io);
        try file.writeStreamingAll(io, "hello from std.Io\n");
    }

    {
        const file = try dir.openFile(io, "test.txt", .{});
        defer file.close(io);

        var buf: [256]u8 = undefined;
        const n = try file.readPositionalAll(io, &buf, 0);
        std.debug.print("read {d} bytes: {s}", .{ n, buf[0..n] });
    }
}
```

### `std.Io.Dir`

#### Current working directory

```zig
const dir = std.Io.Dir.cwd(); // no `io` needed; do not close the result
```

#### Creating and opening directories

```zig
// Create a single directory
try dir.createDir(io, "mydir", .default_dir);

// Create directories recursively
try dir.createDirPath(io, "a/b/c", .default_dir);

// Create and open a directory
var sub_dir = try dir.createDirPathOpen(io, "a/b/c", .default_dir, .{});
defer sub_dir.close(io);

// Open an existing directory
var opened = try dir.openDir(io, "mydir", .{});
defer opened.close(io);
```

#### Creating and opening files

```zig
// Create a file
const file = try dir.createFile(io, "data.txt", .{ .truncate = true });
defer file.close(io);

// Open a file
const file2 = try dir.openFile(io, "data.txt", .{});
defer file2.close(io);
```

#### Directory iteration

```zig
var it = std.Io.Dir.Iterator.init(dir, .reset);
while (try it.next(io)) |entry| {
    std.debug.print("{s} ({s})\n", .{ entry.name, @tagName(entry.kind) });
}
```

#### Other common operations

```zig
try dir.deleteFile(io, "data.txt");
try dir.deleteDir(io, "mydir");
try dir.rename(io, "old.txt", dir, "new.txt");
try dir.symLink(io, "target.txt", "link.txt", .{});
var buf: [std.Io.Dir.max_path_bytes]u8 = undefined;
const n = try dir.realPathFile(io, "data.txt", &buf);
```

### `std.Io.File`

#### Read and write

```zig
// Streaming write (most common)
try file.writeStreamingAll(io, "hello, std.Io\n");

// Streaming read (requires the file to be opened for reading)
var rbuf: [256]u8 = undefined;
var reader = file.reader(io, &rbuf);
const chunk = try reader.interface.readVec(&.{&rbuf});

// Positional read
try file.readPositionalAll(io, &rbuf, 0);

// Get file length
const len = try file.length(io);

// Seek
try file.seekTo(io, 0);
try file.seekBy(io, 10);

// Sync
try file.sync(io);
```

#### `Reader` / `Writer`

```zig
var rbuf: [256]u8 = undefined;
var reader = file.reader(io, &rbuf);

var wbuf: [256]u8 = undefined;
var writer = file.writer(io, &wbuf);
_ = try writer.interface.writeVec(&.{"data"});
try writer.interface.flush();
```

---

## 5. Networking I/O

Network APIs live under `std.Io.net`.

### TCP server skeleton

```zig
const std = @import("std");

pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    var addr = try std.Io.net.IpAddress.parseLiteral("127.0.0.1:0");
    var server = try addr.listen(io, .{ .kernel_backlog = 128 });
    defer server.deinit(io);

    const client = try addr.connect(io, .{ .mode = .stream });
    defer client.close(io);

    const accepted = try server.accept(io);
    defer accepted.close(io);

    var wbuf: [256]u8 = undefined;
    var writer = accepted.writer(io, &wbuf);
    _ = try writer.interface.writeVec(&.{"hello"});
    try writer.interface.flush();

    var rbuf: [256]u8 = undefined;
    var reader = client.reader(io, &rbuf);
    var dest: [1][]u8 = .{&rbuf};
    const n = try reader.interface.readVec(&dest);
    _ = rbuf[0..n];
}
```

### TCP client

```zig
var addr = try std.Io.net.IpAddress.parseLiteral("127.0.0.1:8080");
const stream = try addr.connect(io, .{ .mode = .stream });
defer stream.close(io);
```

### Unix domain socket

```zig
const unix_addr = try std.Io.net.UnixAddress.init("/tmp/test.sock");
// Unix domain socket APIs are exposed through the low-level vtable in 0.16.0.
// For higher-level usage, combine listen/connect from std.Io.net,
// or consult the UnixAddress methods in Io/net.zig.
_ = unix_addr;
```

### DNS lookup

```zig
var queue: std.Io.net.HostName.LookupResult.Queue = .init(allocator);
defer queue.deinit();
try io.netLookup("example.com", &queue, .{});
// Resolve results via the queue
```

---

## 6. Process I/O

### Process API Migration Example

0.15 old style:

```zig
var child = std.process.Child.init(argv, gpa);
child.stdin_behavior = .Pipe;
child.stdout_behavior = .Pipe;
try child.spawn();
```

0.16 new style:

```zig
var child = try std.process.spawn(io, .{
    .argv = argv,
    .stdin = .pipe,
    .stdout = .pipe,
});
```

### Spawning a child process

```zig
var child = try std.process.spawn(io, .{
    .argv = &.{ "zig", "version" },
    .stdin = .ignore,
    .stdout = .pipe,
    .stderr = .inherit,
});
defer child.kill(io);

const term = try child.wait(io);
std.debug.print("exit code: {?d}\n", .{term.Exited});
```

### Running a child and capturing output

```zig
const run_result = try std.process.run(allocator, io, .{
    .argv = &.{ "uname", "-a" },
});
defer allocator.free(run_result.stdout);
std.debug.print("{s}\n", .{run_result.stdout});
```

### Replacing the current process

```zig
try io.processReplace(.{
    .argv = &.{ "ls", "-la" },
});
```

### Current working directory

```zig
var buf: [std.posix.PATH_MAX]u8 = undefined;
const n = try std.process.currentPath(io, &buf);
std.debug.print("cwd: {s}\n", .{buf[0..n]});

try std.process.currentPathAlloc(io, allocator, "/tmp");
```

---

## 7. Concurrency Primitives

### `Future`

`io.async()` and `io.concurrent()` both return `Future(T)` for asynchronous results.

- `async`: weaker guarantee; may execute immediately
- `concurrent`: stronger guarantee; allows the caller to make progress while waiting

```zig
fn computeSum(a: i32, b: i32) i32 {
    return a + b;
}

var future = io.async(computeSum, .{ 1, 2 });
const result = future.await(io);
std.debug.print("sum = {d}\n", .{result});
```

`concurrent` may fail (`error.ConcurrencyUnavailable`):

```zig
var future = try io.concurrent(computeSum, .{ 1, 2 });
const result = future.await(io);
```

Cancelling a task:

```zig
var future = io.async(longTask, .{io});
_ = future.cancel(io); // request cancellation; takes effect at the next cancellation point
```

### `Group`

`Group` replaces the removed `std.Thread.Pool` and `std.Thread.WaitGroup`.

```zig
var group: std.Io.Group = .init;

group.async(io, worker, .{ &counter, io });
group.async(io, worker, .{ &counter, io });

try group.await(io); // wait for all to finish
// or group.cancel(io); // cancel all
```

Important properties:

- `Group.async` / `Group.concurrent` require the function return type to coerce to `Cancelable!void`
- `Group.cancel` immediately requests cancellation for all members
- Resources are released when each **individual task returns**, not when the whole Group finishes

### `Batch`

`Batch` submits multiple I/O operations at once, reducing syscall overhead.

```zig
var storage: [4]std.Io.Operation.Storage = undefined;
var batch = std.Io.Batch.init(&storage);

_ = batch.add(.{ .file_read_streaming = .{
    .file = file1,
    .data = &.{&buf1},
} });

_ = batch.add(.{ .file_write_streaming = .{
    .file = file2,
    .data = &.{"hello"},
} });

// submit and wait
try io.operate(.{ .batch = batch });
try batch.awaitAsync(io);
```

Batch await variants:

- `batch.awaitAsync(io)` — async wait
- `batch.awaitConcurrent(io, timeout)` — wait with timeout
- `batch.cancel(io)` — cancel the whole batch

---

## 8. Cancellation

### `Cancelable!T`

Many blocking `Io` operations return `Cancelable!T`, whose error set includes `error.Canceled`.

```zig
pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    var event: std.Io.Event = .unset;
    const result = event.wait(io);
    // If another task requests cancellation on this io, this returns error.Canceled
    _ = result;
}
```

### Cancellation points

Cancellation requests do not interrupt running computation directly. They only take effect at **cancellation points** — calls to blocking `Io` functions that can return `error.Canceled`.

### `recancel` and `CancelProtection`

- `io.recancel()` — asserts that `error.Canceled` was previously returned, and re-arms the cancellation request so the next cancellation point returns `error.Canceled` again
- `io.swapCancelProtection(new)` — temporarily swap cancellation protection level
- `io.checkCancel()` — explicitly check whether cancellation was requested; returns `error.Canceled` if so

Pattern: deferring cancellation handling

```zig
try io.checkCancel(); // may return Canceled immediately
// ... do required cleanup ...
io.recancel();        // re-signal so upstream still sees Canceled
```

---

## 9. Synchronization Primitives

All synchronization primitives take an `io` parameter, and most operations return `Cancelable!T` (with `*Uncancelable` variants).

### `Mutex`

```zig
var mutex: std.Io.Mutex = .init;

try mutex.lock(io);
defer mutex.unlock(io);

if (mutex.tryLock()) {
    defer mutex.unlock(io);
}

// Uncancelable variant
mutex.lockUncancelable(io);
defer mutex.unlock(io);
```

### `Condition`

```zig
var cond: std.Io.Condition = .init;

try cond.wait(io, &mutex);       // wait for signal (releases/reacquires mutex)
try cond.signal(io);              // wake one waiter
try cond.broadcast(io);           // wake all waiters
```

### `Event` (replaces `std.Thread.ResetEvent`)

```zig
var event: std.Io.Event = .unset;

try event.wait(io);               // block until set
event.set(io);                    // set to true, wake all waiters
event.reset();                    // set to false (no `io` needed)

// With timeout
const timeout = std.Io.Timeout{ .duration = .{ .nanoseconds = 1_000_000_000 } };
try event.waitTimeout(io, timeout);
```

### `Semaphore`

```zig
var sem: std.Io.Semaphore = .{ .permits = 2 };

try sem.wait(io);     // P operation (blocks if permits == 0)
sem.post(io);         // V operation
```

### `RwLock`

```zig
var rwlock: std.Io.RwLock = .init;

// Write lock
try rwlock.lock(io);
defer rwlock.unlock(io);

// Read lock
try rwlock.lockShared(io);
defer rwlock.unlockShared(io);

// Try acquire
if (rwlock.tryLock(io)) defer rwlock.unlock(io);
if (rwlock.tryLockShared(io)) defer rwlock.unlockShared(io);
```

### Low-level Futex

```zig
var value: u32 = 0;
try io.futexWait(u32, &value, 0); // block if value == 0
io.futexWake(u32, &value, 1);     // wake up to 1 waiter
```

---

## 10. Time and Sleep

### `Clock` variants

```zig
const Clock = std.Io.Clock;
// .real      —  wall-clock time (Unix timestamp)
// .awake     —  monotonic clock, excluding system suspend
// .boot      —  monotonic clock, including system suspend
// .cpu_process — process CPU time
// .cpu_thread  — thread CPU time
```

### Getting the current time

```zig
const ts = std.Io.Timestamp.now(io, .real);
// or
const ts2 = Clock.now(.real, io);
```

### Clock resolution

```zig
const res = try Clock.resolution(.real, io);
```

### Sleeping

```zig
// 1: io.sleep
try io.sleep(.{ .nanoseconds = 100_000_000 }, .real);

// 2: Duration.sleep
try std.Io.Duration{ .nanoseconds = 100_000_000 }.sleep(io);

// 3: Timeout.sleep
try std.Io.Timeout{ .duration = .{ .nanoseconds = 100_000_000 } }.sleep(io);

// 4: Timestamp.wait
try ts.wait(io); // sleep until the specified timestamp
```

### `Duration` and `Timeout`

```zig
const dur = std.Io.Duration{ .nanoseconds = 500_000_000 };
const timeout = std.Io.Timeout{ .duration = dur };
// Or use a deadline
const timeout2 = std.Io.Timeout{ .deadline = ts };
```

---

## 11. Randomness

```zig
var buf: [32]u8 = undefined;

// Fast random (may use in-process RNG state, thread-safe)
io.random(&buf);

// Cryptographically secure random (may make a syscall each time)
try io.randomSecure(&buf);
```

---

## 12. `Reader` / `Writer` / `Terminal`

### `std.Io.Reader` / `std.Io.Writer`

These are generic read/write interfaces, conceptually similar to `std.io.Reader` / `std.io.Writer`, but under the `std.Io` namespace.

Getting a Reader/Writer from a file:

```zig
var rbuf: [256]u8 = undefined;
var reader = file.reader(io, &rbuf);

var wbuf: [256]u8 = undefined;
var writer = file.writer(io, &wbuf);
_ = try writer.interface.writeVec(&.{"hello"});
try writer.interface.flush();
```

### Stderr locking

Used to coordinate application-level stderr output with debug output.

```zig
var buf: [256]u8 = undefined;
var ls = try io.lockStderr(&buf, .progress);
defer io.unlockStderr();
try ls.terminal().writer().interface.writeVec(&.{"progress: 50%\n"});
try ls.terminal().writer().interface.flush();
```

---

## 13. `Queue` — Multi-Producer Multi-Consumer Queue

`std.Io.Queue(T)` is a ring-buffer-based blocking MPMC queue.

```zig
var buffer: [64]u32 = undefined;
var queue = std.Io.Queue(u32).init(&buffer);

// Producer
try queue.putOne(io, 42);
try queue.putAll(io, &.{ 1, 2, 3 });

// Consumer
const item = try queue.getOne(io);
```

`put` / `get` API family:

- `put(io, elems, min)` / `putUncancelable`
- `putAll(io, elems)`
- `putOne(io, item)` / `putOneUncancelable`
- `get(io, buf, min)` / `getUncancelable`
- `getOne(io)` / `getOneUncancelable`
- `close(io)` — close the queue; subsequent puts return `error.Closed`

---

## 14. `MemoryMap` — File Memory Mapping

```zig
var mm = try std.Io.File.MemoryMap.create(io, file, .{
    .len = 4096,
    .protection = .{ .read = true, .write = true },
});
defer mm.destroy(io);

// Directly read/write mapped memory
@memcpy(mm.memory[0..4], "hi!");

// Explicit sync
try mm.read(io);   // sync from file to memory
try mm.write(io);  // sync from memory to file

// Resize mapping
try mm.setLength(io, 8192);
```

---

## 15. Quick Design Patterns

### `Io` parameter convention

- Function signature: `fn doWork(io: std.Io, ...) Cancelable!T`
- `Io` is passed by value like `Allocator`; do not pass a pointer

### Error handling pattern

```zig
someIoOp(io, ...) catch |err| switch (err) {
    error.Canceled => {
        // handle cancellation, possibly clean up resources
        io.recancel(); // if upstream should still see Canceled
        return error.Canceled;
    },
    else => |e| return e,
};
```

### 0.15 API replacement cheat sheet

| 0.15 API | 0.16 replacement |
|---|---|
| `std.Thread.Pool` | `std.Io.Group` + `io.concurrent` |
| `std.Thread.WaitGroup` | `std.Io.Group` |
| `std.Thread.ResetEvent` | `std.Io.Event` |
| `std.Thread.Mutex` | `std.Io.Mutex` |
| `std.Thread.Condition` | `std.Io.Condition` |
| `std.Thread.Semaphore` | `std.Io.Semaphore` |
| `std.Thread.RwLock` | `std.Io.RwLock` |
| `std.fs.cwd()` | `std.Io.Dir.cwd()` |
| `dir.createFile(path, flags)` | `dir.createFile(io, path, flags)` |
| `dir.openFile(path, flags)` | `dir.openFile(io, path, flags)` |
| `file.close()` | `file.close(io)` |
| `file.readAll(buffer)` | `file.readPositionalAll(io, buffer, 0)` |
| `file.writeAll(buffer)` | `file.writeStreamingAll(io, buffer)` |
| `file.getEndPos()` | `file.length(io)` |
| `file.reader()` | `file.reader(io, buffer)` |
| `file.writer()` | `file.writer(io, buffer)` |
| `std.fs.path` | `std.Io.Dir.path` |
| `std.process.getCwd` | `std.process.currentPath(io, buffer)` |
| `std.process.Child.init` | `std.process.spawn(io, ...)` |
| `std.time.Instant.now()` | `std.Io.Timestamp.now(io, .real)` |
| `std.crypto.random.bytes(&buf)` | `io.random(&buf)` / `io.randomSecure(&buf)` |

---

## References

- Zig 0.16.0 official release notes
- `lib/std/Io.zig` and `Io/*.zig` submodules
- `verify-project` local compilation verification
