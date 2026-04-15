# @Type to Independent Builtins Migration Guide

## Core Change

Zig 0.16.0 removes the `@Type` builtin and replaces it with 8 independent type-creating builtins.

## Complete Mapping Table

| Old style (`@Type`) | New style (0.16) |
|---|---|
| `@Type(.{ .int = .{ .signedness = .unsigned, .bits = 10 } })` | `@Int(.unsigned, 10)` |
| `@Type(.{ .@"struct" = .{ .layout = .auto, .fields = &[_]std.builtin.Type.StructField{...}, .decls = &.{}, .is_tuple = true } })` | `@Tuple(&.{ u32, [2]f64 })` |
| `@Type(.{ .pointer = .{ .size = .one, .is_const = true, .child = u32, ... } })` | `@Pointer(.one, .{ .@"const" = true }, u32, null)` |
| `@Type(.{ .@"fn" = .{ .calling_convention = .c, .return_type = u32, .params = &.{...} } })` | `@Fn(&.{ f64, *const anyopaque }, &@splat(.{}), u32, .{ .@"callconv" = .c })` |
| `@Type(.{ .@"struct" = .{ .layout = .@"extern", .fields = &.{...}, .decls = &.{}, .is_tuple = false } })` | `@Struct(.@"extern", null, &.{"foo", "bar"}, &.{ [2]f64, u32 }, &@splat(.{}))` |
| `@Type(.{ .@"union" = .{ .layout = .auto, .tag_type = MyEnum, .fields = &.{...}, .decls = &.{}, ... } })` | `@Union(.auto, MyEnum, &.{ "foo", "bar" }, &.{ i64, f64 }, &@splat(.{}))` |
| `@Type(.{ .@"enum" = .{ .tag_type = u32, .fields = &.{...}, .decls = &.{}, .is_exhaustive = true } })` | `@Enum(u32, .exhaustive, &.{ "foo", "bar" }, &.{ 0, 1 })` |
| `@Type(.enum_literal)` | `@EnumLiteral()` |

## Functionality No Longer Provided as Builtins

The following types can no longer be created via builtin; use normal language syntax instead:

- `@Array` → Use `[len]Elem` or `[len:s]Elem` directly
- `@Float` → Use the userland helper `std.meta.Float` (only 5 runtime floating-point types exist)
- `@Opaque` → Use `opaque {}` directly
- `@Optional` → Use `?T` directly
- `@ErrorUnion` → Use `E!T` directly
- `@ErrorSet` → Declare explicitly with `error{ ... }` syntax

## Verified Code Examples

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

## Migration Recommendations

1. Search the whole project for `@Type` and replace each occurrence individually
2. Note that `@Struct` / `@Union` / `@Enum` use a "struct of arrays" style for arguments
3. `&@splat(.{})` is a convenient shorthand for default attributes
