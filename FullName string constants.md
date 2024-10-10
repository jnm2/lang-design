# `FullName` string constants

## Summary

For certain kinds of types, `typeof(`...`).FullName` is considered a constant value. It is allowed anywhere a string constant is allowed, such as in an attribute argument or a constant interpolated string.

## Motivation

The community has requested this feature often. Several duplicates and many variants have been filed, garnering a decent number of upvotes. This would be easy to implement, and it would bring multiple benefits.

First, this change would allow `typeof(`...`).FullName` to be used in places where it is the obvious thing _to_ use, but where it is currently disallowed. You might naturally try to write `[UseThisLogger(typeof(Logger<>).FullName)]`, but today you'll be faced with writing workarounds such as:

```cs
[UseThisLogger(
    nameof(Microsoft) + '.' +
    nameof(Microsoft.Extensions) + '.' +
    nameof(Microsoft.Extensions.Logging) + '.' +
    nameof(Microsoft.Extensions.Logging.Logger<>) + "`1")]
...
```

Second, this change would bring performance improvements in existing code where `typeof(`...`).FullName` is used to build strings. The entire string which includes the type name could become constant, rather than interpolating or concatenating as a runtime operation.

## Detailed design

When `.FullName` is used immediately on a `typeof` expression, and the referenced type is a supported kind of type, the `typeof(`...`).FullName` expression is a string constant. The constant value at compile time to exactly what the expression would evaluate to at runtime.

It will be a constant everywhere, and not just only in places where a constant is required.

### Supported type kinds

Types are supported when they refer to a specific, non-composed, type definition, to obtain the _full name_ of that type definition. This also implies that the resulting strings are known at compile time, which is also a requirement of any type that is supported by this feature.

The _full name_ of a type definition is defined by the runtime's implementation of `System.Type.FullName`. For the types supported by this proposal, the full name generally includes a namespace if any, containing type names if any, and the referenced type name.

Unbound generics are supported. `typeof(List<>).FullName` would produce the constant string ``"System.Collections.Generic.List`1"``.

Nested types are supported. `typeof(List<>.Enumerator).FullName` would produce the constant string ``"System.Collections.Generic.List`1+Enumerator"``.

Primitive type keywords are supported. `typeof(nint).FullName` would produce the constant string `"System.IntPtr"`. This includes `typeof(void).FullName`, which produces `"System.Void"`.

Aliases are supported. The constant string resolves the actual type, the same way the expression would evaluate at runtime.

### Unsupported type kinds

Array types and pointer types are not supported. `typeof(int[])` or `typeof(int*)` or `typeof(int[,][]**)` are composing types over each other, rather than referring to one specific type definition to obtain its namespace and name.

Type parameters are not supported. `typeof(T)` is a type which is not known at compile time.

Bound generics are not supported. `typeof(List<int>).FullName` contains parts known only at runtime. For example: ``"System.Collections.Generic.List`1[[System.Int32, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]"``. The compiler may see only a reference assembly with a different name, or an assembly may load with a different version at runtime than at compile time.

Nullable value types and tuple types are not supported. They fall by both points: they are instances of bound generics, whose `FullName` values can't be known at compile time, and they are also composing multiple type definitions together, contrary to the aim of this proposal which is to identify the full name of a specific type definition.

- `typeof(int?).FullName` produces a string such as: ``"System.Nullable`1[[System.Int32, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]"``.

- `typeof((int, string)).FullName` produces a string such as: ``"System.ValueTuple`2[[System.Int32, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e],[System.String, System.Private.CoreLib, Version=8.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e]]"``.

### Nullability

The `FullName` property on `System.Type` is annotated as nullable, but when `typeof(Xyz).FullName` is a constant, it is never null. When this expression is constant, the compiler's nullability analysis should report it as not-null. This avoids the inevitable need for `!` each time this feature is used with an attribute, since it's not likely that the attribute's type name argument is an optional one.

This will also be a benefit in places that use `typeof(Xyz).FullName!` today.

## Possible extensions

`typeof(Xyz).Name` and `typeof(Xyz).Namespace` are components of the full name, and could be supported in the same way. Unlike `FullName` and `Name`, `Namespace` could evaluate to a constant null string.

`typeof(Xyz<>).Name` differs from `nameof`. Whereas `nameof` returns the C# language name of the type or alias, `typeof(Xyz<>).Name` returns the name in metadata, such as ``"Xyz`1"``.

`typeof(Xyz).Namespace` would make it less of a hassle to refer to a full namespace. Currently, the only resort is something like:

```cs
nameof(System) + '.' +
nameof(System.Collections) + '.' +
nameof(System.Collections.Generic)
```

## Alternatives

Some of the variants of this request are for a `fullnameof` or `pathof` operator. The output would diverge, since `nameof` today returns the C# name of the type or alias being used while `typeof(`...`).Name` returns the metadata name, suitable for reflection. Metadata names use backticks in generic type names and the `+` separator between containing types and nested types. This same distinction would presumably be maintained between `fullnameof` and `typeof(`...`).FullName`.

`fullnameof` could reference namespaces without requiring the naming of a type within the namespace.

`fullnameof` would also be extendable to reference a member, and this is sometimes requested. However, referencing a member along with its containing type name and full namespace seems like specialized use, producing strings which are not usable with any core .NET API. This may be more of a job for an `infoof` operator, providing a reference to the member and letting it be examined at runtime to produce policy-specific formatting suited to the use case.
