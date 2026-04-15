# @Type 替换为独立 Builtins 迁移指南

## 核心变化

Zig 0.16.0 移除了 `@Type` builtin，改为 8 个独立的类型创建 builtins。

## 完整对照表

| 旧写法 (`@Type`) | 新写法 (0.16) |
|---|---|
| `@Type(.{ .int = .{ .signedness = .unsigned, .bits = 10 } })` | `@Int(.unsigned, 10)` |
| `@Type(.{ .@"struct" = .{ .layout = .auto, .fields = &[_]std.builtin.Type.StructField{...}, .decls = &.{}, .is_tuple = true } })` | `@Tuple(&.{ u32, [2]f64 })` |
| `@Type(.{ .pointer = .{ .size = .one, .is_const = true, .child = u32, ... } })` | `@Pointer(.one, .{ .@"const" = true }, u32, null)` |
| `@Type(.{ .@"fn" = .{ .calling_convention = .c, .return_type = u32, .params = &.{...} } })` | `@Fn(&.{ f64, *const anyopaque }, &@splat(.{}), u32, .{ .@"callconv" = .c })` |
| `@Type(.{ .@"struct" = .{ .layout = .@"extern", .fields = &.{...}, .decls = &.{}, .is_tuple = false } })` | `@Struct(.@"extern", null, &.{"foo", "bar"}, &.{ [2]f64, u32 }, &@splat(.{}))` |
| `@Type(.{ .@"union" = .{ .layout = .auto, .tag_type = MyEnum, .fields = &.{...}, .decls = &.{}, ... } })` | `@Union(.auto, MyEnum, &.{ "foo", "bar" }, &.{ i64, f64 }, &@splat(.{}))` |
| `@Type(.{ .@"enum" = .{ .tag_type = u32, .fields = &.{...}, .decls = &.{}, .is_exhaustive = true } })` | `@Enum(u32, .exhaustive, &.{ "foo", "bar" }, &.{ 0, 1 })` |
| `@Type(.enum_literal)` | `@EnumLiteral()` |

## 不再提供的功能

以下类型不再能通过 builtin 创建，需直接使用语言语法：

- `@Array` → 直接使用 `[len]Elem` 或 `[len:s]Elem`
- `@Float` → 使用 `std.meta.Float` 用户态 helper（只有 5 个运行时浮点类型）
- `@Opaque` → 直接使用 `opaque {}`
- `@Optional` → 直接使用 `?T`
- `@ErrorUnion` → 直接使用 `E!T`
- `@ErrorSet` → 直接使用 `error{ ... }` 语法显式声明

## 验证通过的代码示例

```zig
test "@Int works" {
    const T = @Int(.unsigned, 16);
    try @import("std").testing.expect(T == u16);
}

test "@Tuple works" {
    const T = @Tuple(&.{ u32, f64 });
    try @import("std").testing.expect(@typeInfo(T).@"struct".is_tuple);
}

test "@EnumLiteral works" {
    const T = @EnumLiteral();
    try @import("std").testing.expect(T == @TypeOf(.foo));
}

test "@Pointer works" {
    const T = @Pointer(.one, .{ .@"const" = true }, u8, null);
    try @import("std").testing.expect(T == *const u8);
}

test "@Struct works" {
    const T = @Struct(.auto, null, &.{"x", "y"}, &.{ u32, f64 }, &@splat(.{}));
    try @import("std").testing.expect(@sizeOf(T) == @sizeOf(struct { x: u32, y: f64 }));
}

test "@Union works" {
    const T = @Union(.auto, null, &.{"x", "y"}, &.{ u32, f64 }, &@splat(.{}));
    try @import("std").testing.expect(@sizeOf(T) == @sizeOf(union { x: u32, y: f64 }));
}

test "@Enum works" {
    const T = @Enum(u8, .exhaustive, &.{"a", "b"}, &.{ 0, 1 });
    try @import("std").testing.expect(@typeInfo(T).@"enum".tag_type == u8);
}

test "@Fn works" {
    const T = @Fn(&.{ u32, f64 }, &@splat(.{}), void, .{});
    const Expected = fn (u32, f64) void;
    try @import("std").testing.expect(T == Expected);
}
```

## 迁移建议

1. 全项目搜索 `@Type`，逐条替换
2. 注意 `@Struct` / `@Union` / `@Enum` 使用 "struct of arrays" 风格传参
3. `&@splat(.{})` 是设置默认 attributes 的快捷写法
