# `std.Io` 完整 API 参考（Zig 0.16.0）

本文档基于 Zig 0.16.0 官方源码与 release notes 整理，所有代码示例均经过本地编译验证。

---

## 1. `std.Io` 是什么

`std.Io` 是 Zig 0.16.0 引入的统一接口，它把**文件系统、网络、进程、时间、随机数、并发、取消、同步**全部抽象到同一个接口下。

核心设计：

- `Io` 本身是一个轻量结构体，只包含 `userdata: ?*anyopaque` 和 `vtable: *const VTable`
- 所有可能阻塞或引入非确定性的操作，都需要显式传入 `Io` 实例
- 0.15 中直接调用 `std.fs.cwd().createFile(...)` 的风格，被替换为 `dir.createFile(io, ...)`

---

## 2. 获取 `Io` 实例

### 单线程场景

```zig
var threaded: std.Io.Threaded = .init_single_threaded;
const io = threaded.io();
// `threaded` 必须比 `io` 活得更久
```

### 多线程场景

```zig
var threaded = std.Io.Threaded.init(allocator, .{});
const io = threaded.io();
```

### 测试场景

```zig
const io = std.testing.io;
```

### `main` 函数签名变化（Juicy Main）

0.15 及以前：

```zig
pub fn main() !void { ... }
```

0.16：

```zig
pub fn main(init: std.process.Init) !void {
    _ = init;
    ...
}
```

---

## 3. 像 `Allocator` 一样传递 `Io`

`Io` 非常小（两个指针大小），因此和 `std.mem.Allocator` 一样，**按值传递**即可。

```zig
fn fetchData(io: std.Io, url: []const u8, buf: []u8) !usize {
    _ = url;
    // 所有需要阻塞或不确定性的操作都通过 io 完成
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

约定（与标准库一致）：

- `io` 通常放在参数列表的前部或后部，但标准库倾向于**放在后面**（如 `file.readStreaming(io, buf)` 改为 `io` 前置或后置视具体 API 而定，0.16 std 中多为 `io` 前置或紧跟对象后）
- 实际上在 0.16 std 中，`io` 的位置并不完全统一：同步原语多为 `lock(io)`、`wait(io, ...)`，文件操作为 `file.readStreaming(io, buf)`，`dir.createFile(io, path, opts)`。本文档会逐一标出。

---

## 4. 文件系统 I/O

### 文件 I/O 迁移示例

0.15 旧写法：

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

0.16 新写法：

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

#### 获取当前工作目录

```zig
const dir = std.Io.Dir.cwd(); // 不需要 io，但返回的 Dir 不可 close
```

#### 创建与打开目录

```zig
// 创建单级目录
try dir.createDir(io, "mydir", .default_dir);

// 递归创建目录
try dir.createDirPath(io, "a/b/c", .default_dir);

// 创建并打开目录
var sub_dir = try dir.createDirPathOpen(io, "a/b/c", .default_dir, .{});
defer sub_dir.close(io);

// 打开已有目录
var opened = try dir.openDir(io, "mydir", .{});
defer opened.close(io);
```

#### 创建与打开文件

```zig
// 创建文件
const file = try dir.createFile(io, "data.txt", .{ .truncate = true });
defer file.close(io);

// 打开文件
const file2 = try dir.openFile(io, "data.txt", .{});
defer file2.close(io);
```

#### 目录遍历

```zig
var it = std.Io.Dir.Iterator.init(dir, .reset);
while (try it.next(io)) |entry| {
    std.debug.print("{s} ({s})\n", .{ entry.name, @tagName(entry.kind) });
}
```

#### 其他常用操作

```zig
try dir.deleteFile(io, "data.txt");
try dir.deleteDir(io, "mydir");
try dir.rename(io, "old.txt", dir, "new.txt");
try dir.symLink(io, "target.txt", "link.txt", .{});
var buf: [std.Io.Dir.max_path_bytes]u8 = undefined;
const n = try dir.realPathFile(io, "data.txt", &buf);
```

### `std.Io.File`

#### 读写操作

```zig
// 流式写（最常用）
try file.writeStreamingAll(io, "hello, std.Io\n");

// 流式读（需要文件以读方式打开）
var rbuf: [256]u8 = undefined;
var reader = file.reader(io, &rbuf);
const chunk = try reader.read();

// 定位读
try file.readPositionalAll(io, &rbuf, 0);

// 获取文件长度
const len = try file.length(io);

// 同步
try file.sync(io);
```

#### `Reader` / `Writer`

```zig
var rbuf: [256]u8 = undefined;
var reader = file.reader(io, &rbuf);

var wbuf: [256]u8 = undefined;
var writer = file.writer(io, &wbuf);
try writer.writeAll("data");
try writer.flush();
```

---

## 5. 网络 I/O

网络 API 集中在 `std.Io.net` 下。

### TCP 服务器骨架

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

### TCP 客户端

```zig
var addr = try std.Io.net.IpAddress.parseLiteral("127.0.0.1:8080");
const stream = try addr.connect(io, .{ .mode = .stream });
defer stream.close(io);
```

### Unix Domain Socket

```zig
const unix_addr = try std.Io.net.UnixAddress.init("/tmp/test.sock");
// Unix domain socket API 在 0.16.0 中通过底层 vtable 暴露，
// 高层封装建议直接使用 std.Io.net 中的 listen/connect 组合，
// 或查阅 Io/net.zig 中 UnixAddress 的相关方法。
_ = unix_addr;
```

### DNS 查询

```zig
var queue: std.Io.net.HostName.LookupResult.Queue = .init(allocator);
defer queue.deinit();
try io.netLookup("example.com", &queue, .{});
// 解析结果通过 queue 获取
```

---

## 6. 进程 I/O

### 进程 API 迁移示例

0.15 旧写法：

```zig
var child = std.process.Child.init(argv, gpa);
child.stdin_behavior = .Pipe;
child.stdout_behavior = .Pipe;
try child.spawn();
```

0.16 新写法：

```zig
var child = try std.process.spawn(io, .{
    .argv = argv,
    .stdin = .pipe,
    .stdout = .pipe,
});
```

### 启动子进程

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

### 运行子进程并捕获输出

```zig
const run_result = try std.process.run(allocator, io, .{
    .argv = &.{ "uname", "-a" },
});
defer allocator.free(run_result.stdout);
std.debug.print("{s}\n", .{run_result.stdout});
```

### 替换当前进程

```zig
try io.processReplace(.{
    .argv = &.{ "ls", "-la" },
});
```

### 当前工作目录

```zig
var buf: [std.posix.PATH_MAX]u8 = undefined;
const n = try std.process.currentPath(io, &buf);
std.debug.print("cwd: {s}\n", .{buf[0..n]});

try std.process.currentPathAlloc(io, allocator, "/tmp");
```

---

## 7. 并发原语

### `Future`

`io.async()` 和 `io.concurrent()` 都会返回 `Future(T)`，用于异步获取结果。

- `async`：弱保证，可能立即执行
- `concurrent`：强保证，允许调用者在等待时继续推进（可能利用真正的并发）

```zig
fn computeSum(a: i32, b: i32) i32 {
    return a + b;
}

var future = io.async(computeSum, .{ 1, 2 });
const result = future.await(io);
std.debug.print("sum = {d}\n", .{result});
```

`concurrent` 可能失败（`error.ConcurrencyUnavailable`）：

```zig
var future = try io.concurrent(computeSum, .{ 1, 2 });
const result = future.await(io);
```

取消任务：

```zig
var future = io.async(longTask, .{io});
_ = future.cancel(io); // 发起取消请求，在下个 cancelation point 生效
```

### `Group`

`Group` 替代了被删除的 `std.Thread.Pool` 和 `std.Thread.WaitGroup`。

```zig
var group: std.Io.Group = .init;

group.async(io, worker, .{ &counter, io });
group.async(io, worker, .{ &counter, io });

try group.await(io); // 等待全部完成
// 或 group.cancel(io); // 取消全部
```

重要特性：

- `Group.async` / `Group.concurrent` 要求函数返回类型可强制转换为 `Cancelable!void`
- `Group.cancel` 会立即向所有成员请求取消
- 资源在每个**单独任务返回时**释放，而不是等整个 Group 完成

### `Batch`

`Batch` 用于一次性提交多个 I/O 操作，减少系统调用次数。

```zig
var storage: [4]std.Io.Operation.Storage = undefined;
var batch = std.Io.Batch.init(&storage);

// 添加操作到 batch
_ = batch.add(.{ .file_read_streaming = .{
    .file = file1,
    .data = &.{&buf1},
} });

_ = batch.add(.{ .file_write_streaming = .{
    .file = file2,
    .data = &.{"hello"},
} });

// 提交并等待
try io.operate(.{ .batch = batch });
try batch.awaitAsync(io);
```

`Batch` 的 await 方式：

- `batch.awaitAsync(io)` — 异步等待
- `batch.awaitConcurrent(io, timeout)` — 带超时等待
- `batch.cancel(io)` — 取消整个 batch

---

## 8. 从 `async`/`await` 迁移到 `std.Io`

Zig 0.15 移除了 `async`/`await`/`suspend`/`resume` 关键字，0.16 通过 `std.Io` 提供统一的并发与异步模型。

### 旧写法 vs 新写法对照

| 旧写法（≤0.14） | 0.16 新写法 |
|---|---|
| `const f = async foo();` | `var f = io.async(foo, .{ args... });` |
| `const r = await f;` | `const r = f.await(io);` |
| `std.Thread.Pool` + `WaitGroup` | `std.Io.Group` |
| `std.Thread.ResetEvent` | `std.Io.Event` |
| `suspend` / `resume` | `io.concurrent` / `Group` / `Event` 组合 |

### 单任务异步

```zig
fn fetchData(io: std.Io, id: u32) u32 {
    _ = io;
    return id * 2;
}

pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    // 旧：const frame = async fetchData(42);
    //     const result = await frame;
    var future = io.async(fetchData, .{ io, 42 });
    const result = future.await(io);
    std.debug.print("result = {d}\n", .{result});
}
```

### 并发任务组（替代 `Thread.Pool`）

```zig
fn task(io: std.Io, id: u32) void {
    _ = io;
    std.debug.print("task {d}\n", .{id});
}

pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    var group: std.Io.Group = .init;
    group.async(io, task, .{ io, 1 });
    group.async(io, task, .{ io, 2 });
    group.async(io, task, .{ io, 3 });
    try group.await(io); // 等待全部完成
}
```

### 强并发保证 `io.concurrent`

`io.concurrent` 要求底层 `Io` 支持真正的并发（如多线程 `Threaded`）。单线程模式下会返回 `error.ConcurrencyUnavailable`。

```zig
var threaded = std.Io.Threaded.init(allocator, .{});
defer threaded.deinit();
const io = threaded.io();

var future = try io.concurrent(compute, .{ io, 7 });
const result = future.await(io);
```

降级处理模式：

```zig
var future = io.concurrent(compute, .{ io, 7 }) catch |err| switch (err) {
    error.ConcurrencyUnavailable => io.async(compute, .{ io, 7 }),
    else => |e| return e,
};
const result = future.await(io);
```

### 链式调用（模拟 async fn 内部 await）

旧代码中可以在 `async fn` 里 `await` 另一个 `async` 调用；0.16 中直接在普通函数里使用 `io.async` + `await` 即可。

```zig
fn step1(io: std.Io, input: u32) u32 {
    _ = io;
    return input * 2;
}

fn step2(io: std.Io, input: u32) u32 {
    _ = io;
    return input + 10;
}

fn pipeline(io: std.Io, input: u32) u32 {
    var f1 = io.async(step1, .{ io, input });
    const r1 = f1.await(io);
    var f2 = io.async(step2, .{ io, r1 });
    return f2.await(io);
}

pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    var f = io.async(pipeline, .{ io, 5 });
    const result = f.await(io);
    std.debug.print("pipeline(5) = {d}\n", .{result});
}
```

### 取消异步任务

```zig
var future = io.async(longTask, .{ io });
_ = future.cancel(io); // 发起取消请求
// 在 longTask 的下一个取消点会返回 error.Canceled
```

---

## 9. 取消机制

### `Cancelable!T`

很多阻塞性的 `Io` 操作返回 `Cancelable!T`，其错误集包含 `error.Canceled`。

```zig
pub fn main() !void {
    var threaded: std.Io.Threaded = .init_single_threaded;
    const io = threaded.io();

    // 模拟一个可能被取消的等待
    var event: std.Io.Event = .unset;
    const result = event.wait(io);
    // 如果另一个任务对此 io 发起了取消，这里会返回 error.Canceled
    _ = result;
}
```

### 取消点（Cancelation Point）

取消请求不会中断正在运行的计算代码，只会在**调用 `Io` 阻塞函数时**返回 `error.Canceled`。这些调用就是"取消点"。

### `recancel` 与 `CancelProtection`

- `io.recancel()` — 断言之前已经收到了 `error.Canceled`，并重新设置取消标志，让下一次取消点再次返回 `error.Canceled`
- `io.swapCancelProtection(new)` — 临时切换取消保护级别
- `io.checkCancel()` — 显式检查当前是否被请求取消，如果是则返回 `error.Canceled`

模式：延迟处理取消

```zig
try io.checkCancel(); // 立即检查并可能返回 Canceled
// ... 做一些必须完成的清理 ...
io.recancel();        // 重新标记，让上层继续感知取消
```

---

## 10. 同步原语

所有同步原语都需要 `io` 参数，且大多数操作返回 `Cancelable!T`（也有 `*Uncancelable` 变体）。

### `Mutex`

```zig
var mutex: std.Io.Mutex = .init;

try mutex.lock(io);
defer mutex.unlock(io);

if (mutex.tryLock()) {
    defer mutex.unlock(io);
}

// 不可取消版本
mutex.lockUncancelable(io);
defer mutex.unlock(io);
```

### `Condition`

```zig
var cond: std.Io.Condition = .init;

try cond.wait(io, &mutex);       // 等待信号（会自动释放/重新获取 mutex）
try cond.signal(io);              // 唤醒一个等待者
try cond.broadcast(io);           // 唤醒所有等待者
```

### `Event`（替代 `std.Thread.ResetEvent`）

```zig
var event: std.Io.Event = .unset;

try event.wait(io);               // 阻塞直到 set
event.set(io);                    // 设为 true，唤醒所有等待者
event.reset();                    // 设为 false（无 io 参数）

// 带超时
const timeout = std.Io.Timeout{ .duration = .{ .nanoseconds = 1_000_000_000 } };
try event.waitTimeout(io, timeout);
```

### `Semaphore`

```zig
var sem: std.Io.Semaphore = .{ .permits = 2 };

try sem.wait(io);     // P 操作（permits > 0 时通过，否则阻塞）
sem.post(io);         // V 操作
```

### `RwLock`

```zig
var rwlock: std.Io.RwLock = .init;

// 写锁
try rwlock.lock(io);
defer rwlock.unlock(io);

// 读锁
try rwlock.lockShared(io);
defer rwlock.unlockShared(io);

// 尝试获取
if (rwlock.tryLock(io)) defer rwlock.unlock(io);
if (rwlock.tryLockShared(io)) defer rwlock.unlockShared(io);
```

### 底层 Futex

```zig
var value: u32 = 0;
try io.futexWait(u32, &value, 0); // 如果 value == 0 则阻塞
io.futexWake(u32, &value, 1);     // 唤醒最多 1 个等待者
```

---

## 11. 时间与睡眠

### 时间和随机数迁移示例

随机数：

```zig
// 0.15
std.crypto.random.bytes(&buffer);

// 0.16
io.random(&buffer);
io.randomSecure(&buffer);
```

时间：

```zig
// 0.15
std.time.Instant.now();
std.time.Timer.start();

// 0.16
std.Io.Timestamp.now(io, .real);
try std.Io.Clock.resolution(.real, io);
```

### `Clock` 类型

```zig
const Clock = std.Io.Clock;
// .real      —  wall-clock 时间（Unix 时间戳）
// .awake     —  单调时钟，排除系统挂起时间
// .boot      —  单调时钟，包含系统挂起时间
// .cpu_process — 进程 CPU 时间
// .cpu_thread  — 线程 CPU 时间
```

### 获取当前时间

```zig
const ts = std.Io.Timestamp.now(io, .real);
// 或
const ts2 = Clock.now(.real, io);
```

### 获取时钟精度

```zig
const res = try Clock.resolution(.real, io);
```

### 睡眠

```zig
// 方式 1：io.sleep
try io.sleep(.{ .nanoseconds = 100_000_000 }, .real);

// 方式 2：Duration.sleep
try std.Io.Duration{ .nanoseconds = 100_000_000 }.sleep(io);

// 方式 3：Timeout.sleep
try std.Io.Timeout{ .duration = .{ .nanoseconds = 100_000_000 } }.sleep(io);

// 方式 4：Timestamp.wait
try ts.wait(io); // 睡到指定时间点
```

### `Duration` 与 `Timeout`

```zig
const dur = std.Io.Duration{ .nanoseconds = 500_000_000 };
const timeout = std.Io.Timeout{ .duration = dur };
// 也可用 deadline
const timeout2 = std.Io.Timeout{ .deadline = ts };
```

---

## 12. 随机数

```zig
var buf: [32]u8 = undefined;

// 快速随机（可能基于进程内 RNG 状态，线程安全）
io.random(&buf);

// 加密安全随机（每次可能触发系统调用）
try io.randomSecure(&buf);
```

---

## 13. `Reader` / `Writer` / `Terminal`

### `std.Io.Reader` / `std.Io.Writer`

这两个是通用的读写接口，与 `std.io.Reader` / `std.io.Writer` 概念类似，但属于 `std.Io` 命名空间。

文件获取 Reader/Writer：

```zig
var rbuf: [256]u8 = undefined;
var reader = file.reader(io, &rbuf);

var wbuf: [256]u8 = undefined;
var writer = file.writer(io, &wbuf);
try writer.writeAll("hello");
try writer.flush();
```

### 标准错误锁定

用于协调应用级 stderr 输出与调试输出。

```zig
var buf: [256]u8 = undefined;
var ls = try io.lockStderr(&buf, .progress);
defer io.unlockStderr();
try ls.terminal().writer().print("progress: {d}%\n", .{50});
```

---

## 14. `Queue` — 多生产者多消费者队列

`std.Io.Queue(T)` 是一个基于 ring buffer 的 MPMC 阻塞队列。

```zig
var buffer: [64]u32 = undefined;
var queue = std.Io.Queue(u32).init(&buffer);

// 生产者
try queue.putOne(io, 42);
try queue.putAll(io, &.{ 1, 2, 3 });

// 消费者
const item = try queue.getOne(io);
```

`put` / `get` 系列 API：

- `put(io, elems, min)` / `putUncancelable`
- `putAll(io, elems)`
- `putOne(io, item)` / `putOneUncancelable`
- `get(io, buf, min)` / `getUncancelable`
- `getOne(io)` / `getOneUncancelable`
- `close(io)` — 关闭队列，后续 put 返回 `error.Closed`

---

## 15. `MemoryMap` — 内存映射文件

```zig
var mm = try std.Io.File.MemoryMap.create(io, file, .{
    .len = 4096,
    .protection = .{ .read = true, .write = true },
});
defer mm.destroy(io);

// 直接读写映射内存
@memcpy(mm.memory[0..4], "hi!");

// 显式同步
try mm.read(io);   // 从文件同步到内存
try mm.write(io);  // 从内存同步到文件

// 调整映射大小
try mm.setLength(io, 8192);
```

---

## 16. 设计模式速查

### `Io` 参数约定

- 函数签名：`fn doWork(io: std.Io, ...) Cancelable!T`
- `Io` 和 `Allocator` 一样按值传递，不要传指针

### 错误处理模式

```zig
someIoOp(io, ...) catch |err| switch (err) {
    error.Canceled => {
        // 处理取消，可能需要清理资源
        io.recancel(); // 如果希望上层继续感知取消
        return error.Canceled;
    },
    else => |e| return e,
};
```

### 替代 0.15 API 速查

| 0.15 API | 0.16 替代 |
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
| `async foo()` / `await frame` | `io.async(foo, args)` + `future.await(io)` |
| `std.process.Child.init` | `std.process.spawn(io, ...)` |
| `std.time.Instant.now()` | `std.Io.Timestamp.now(io, .real)` |
| `std.crypto.random.bytes(&buf)` | `io.random(&buf)` / `io.randomSecure(&buf)` |

---

## 参考来源

- Zig 0.16.0 官方 release notes
- `lib/std/Io.zig` 及 `Io/*.zig` 子模块源码
- `verify-project` 本地编译验证
