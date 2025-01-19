# Target-typed static member lookup

Champion issue: <TODO>

## Summary

This feature enables a type name to be omitted from static member access when it is the same as the target type.

This reduces construction and consumption verbosity for factory methods, nested derived types, enum values, constants, singletons, and other static members. By doing so, the way is also paved for discriminated unions to benefit from the same concise construction and consumption syntaxes.

Examples:

```cs
type.GetMethod("Name", .Public | .Instance | .DeclaredOnly); // BindingFlags.Public | ...

control.ForeColor = .Red;          // Color.Red
entity.InvoiceDate = .Now;         // DateTime.Now
ReadJsonDocument(.Parse(stream));  // JsonDocument.Parse

// Production (static members on Option<int>)
Option<int> option = condition ? .None : .Some(42);

// Production (nested derived types)
CustomResult result = condition ? new .Success(42) : new .Error("message");

// Consumption (nested derived types)
return result switch
{
    .Success(var val) => val,
    .Error => defaultVal,
};
```

## Motivation

LDM prior interest <https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-09-26.md#discriminated-unions>

DUs, existing nested type checks, BindingFlags

## Detailed design

1. In target-typed locations, a new primary expression may be used which starts with `.` and is followed by an identifier.

2. Target-typed locations are expanded to include operands of overloadable operators, and expressions that are invoked.

### Target typing through overloadable operators

TODO: flesh out. Consider unary (`~.None`), non-bitwise (`new(1) + new(2)`)

### Notes

As with target-typed `new`, targeting a nullable value type should access members on the inner value type:

```cs
Point? p = new();  // Equivalent to: new Point()
Point? p = .Empty; // Equivalent to: Point.Empty
```

As with target-typed `new`, overload resolution is not influenced by the presence of a target-typed static member expression. If overload resolution was influenced, it would become a breaking change to add any new static member to a type.

```cs
M(.Empty); // Overload ambiguity error

void M(string p) { }
void M(object p) { }
```

## Specification

`'.' identifier type_argument_list?` is consolidated into a standalone syntax, `member_binding`, and this new syntax is added as a production of the [§12.8.7 Member access](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/expressions.md#1287-member-access) grammar:

```diff
 member_access
-    : primary_expression '.' identifier type_argument_list?
-    | predefined_type '.' identifier type_argument_list?
-    | qualified_alias_member '.' identifier type_argument_list?
+    : primary_expression member_binding
+    | predefined_type member_binding
+    | qualified_alias_member member_binding
+    | member_binding
     ;

+member_binding
+    '.' identifier type_argument_list?

 base_access
-    : 'base' '.' identifier type_argument_list?
+    : 'base' member_binding
     | 'base' '[' argument_list ']'
     ;

 null_conditional_member_access
-    : primary_expression '?' '.' identifier type_argument_list?
+    : primary_expression '?' member_binding
       (null_forgiving_operator? dependent_access)*
     ;

 dependent_access
-    : '.' identifier type_argument_list?    // member access
+    : member_binding            // member access
     | '[' argument_list ']'     // element access
     | '(' argument_list? ')'    // invocation
     ;

 null_conditional_projection_initializer
-    : primary_expression '?' '.' identifier type_argument_list?
+    : primary_expression '?' member_binding
     ;
```

As a primary expression, `member_binding` is only permitted in locations which provide a target type.

TODO: flesh out. Expand target-typing through overloadable operators and invocation expressions. Compare to how target-typing works for `??`.

### Further spec simplification

TODO: Flesh out. Introduce `binding`, which is either of `member_binding`, or `element_binding` (with `'['`)

## Limitations

One of the use cases this feature serves is production and consumption of values of nested derived types, for discriminated unions and other scenarios. But one consumption scenario that is left out of this improvement is `results.OfType<.Error>()`. It's not possible to target-type in this location because the `T` is not correlated with `results`. This problem would likely only be solvable in a general way with annotations that would need to ship with the `OfType` declaration.

A new operator could solve this, such as `results.SelectNonNull(r => r as .Error?)`.

## Drawbacks

### Ambiguities

There are a couple of ambiguities, with [parenthesized expressions](#ambiguity-with-parenthesized-expression) and [conditional expressions](#ambiguity-with-conditional-expression). See each link for details.

### Factory methods public in generic types

The availability of this feature will flip a current framework design guideline on its head. Currently, the design guideline is to declare a static nongeneric class with a generic helper method so that inference is possible: `ImmutableArray.Create<T>`, not `ImmutableArray<T>.Create`. When people declare Option types, it's similarly `Option.Some<T>`, not `Option<T>.Some`.

When target-typing `Option<int> opt = .Some(42)`, what will be called is a static method on the `Option<T>` type rather than on a static helper `Option` type. This will require library authors to provide public factory methods in both places, if they want to cater to both target-typed construction (`.Some(42)`) and to non-target-typed inference (`var opt = Option.Some(42);`).

## Anti-drawbacks

There's been a separate request to mimic VB's `With` construct, allowing dotted access to the expression anywhere within the block:

```cs
with (expr)
{
    .PropOnExpr = 42;
    list.Add(.PropOnExpr);
}
```

This doesn't seem to be a popular request among the language team members who have commented on this. If we go ahead with the proposed target-typing for `.Name` syntax, this seals the fate of the requested `with` statement syntax shown here.

## Expansions

To match the production and consumption sides even better, it could be very desirable to enable the `new` operator to look up nested derived types in the same way:

```cs
new .Case1(arg1, arg2)
```

This would continue to be target-typed static member access (since nested types are members of their containing type), which is distinct from target-typed `new` since a definite type is provided to the `new` operator.

If target-typed static member access is not allowed in this location, the downside is that the production and consumption syntaxes will not have parity.

```cs
du switch
{
    .Case1(var arg1, var arg2) => ...
}
```

In cases where the union name is long, perhaps `SomeDiscriminatedUnion<ImmutableArray<int>>`, this will really stand out.

## Alternatives

### Alternative: doing nothing

Generally speaking, production and consumption of discriminated union values will be fairly onerous as mentioned in the [Motivation](#motivation) section, e.g. having to write `is Option<ImmutableArray<Xyz>>.None` rather than `is .None`.

#### Workaround: `using static`

As a mitigation, `using static` directives can be applied as needed at the top of the file or globally. This allows syntax such as `GetMethod("Name", Public | Static)` today.

This comes with a severe limitation, however, in that it doesn't help much with generic types. If you import `Option<int>`, you can write `is Some`, but only for `Option<int>` and not `Option<string>` or any other constructed type.

Secondly, the `using static` workaround suffers from lack of precedence. Anything member in scope with the same name takes precedence over the member you're trying to access. The proposed syntax for this feature solves this with the leading `.`, which unambiguously shows that the identifier that follows comes from the target type, and not from the current scope.

Third, the `using static` workaround is an imprecise hammer. It puts the names in scope in places where you might not want them. Imagine `var materialType = Slate;`: Maybe you thought this was an enum value in your roofing domain, but accidentally picked up a web color instead.

The `using static` approach has also not found broad adoption over fully qualifying. There are very low hit counts on grep.app and github.com for `using static System.Reflection.BindingFlags`.

### Alternative: no sigil

Target-typed static member lookup benefits from the precision of the `.` sigil, but it does not require a sigil. `GetMethod("Name", Public | Static)` could be the chosen syntax. However, a sigil is strongly recommended for two reasons: user comprehension, and power.

The feature would be harder to understand without a sigil. When target-typed names are added into the universe of available names to look up from, it's almost like tearing a wormhole in spacetime. It's a powerful event. The lookup rules change behavior. That's a good match for new syntax indicating "the rules are different here." You are traversing a wormhole to a different lookup universe. This is making the feature stand out, but not for the sake of standing out as a new feature. The places where the rules behave differently should be coupled with a visible, though still minimal, marker, or essential context is missing. If no such marker is in place, it will slow down understanding of code. Every identifier will need to be considered as to whether it is in a target-typing location and could be referring to something on that type. The chance of collisions is expected to be high. The presence of `.` makes reading much more efficient.

The feature would become less powerful without a sigil. To avoid changes in meaning, this would have to prefer binding to other things in the current scope name, with target-typing as a fallback. This would result in unpleasant interruptions with no recourse other than typing out the full type name. These interruptions are expected to be frequent enough to hamper the success of the feature.

## Open questions

### Ambiguity with parenthesized expression

This is valid grammar today, which fails in binding if `A` is a type and not a value: `(A).B`.

The new grammar we're adding would allow this to be parsed as a cast followed by a target-typed static member lookup. This new interpretation is consistent with `(A)new()` and `(A)default` working today, but it would not be practically useful. `A.B` is a simpler and clearer way to write the same thing.

Should `(A).B` continue to fail, or be made to work the same as `A.B` when `A` is a type?

### Ambiguity with conditional expression

There is an ambiguity if target-typed static member lookup is used as the first branch of a conditional expression, where it would parse today as a null-safe dereference: `expr ? .Name : ...`

We can follow the approach already taken for the similar ambiguity in collection expressions with `expr ? [` possibly being an indexer and possibly being a collection expression.

Alternatively, target-typed static member lookup could be always disallowed within the first branch of a conditional expression unless surrounded by parens: `expr ? (.Name) : ...`. The downside is that this puts a usability burden onto users, since the compiler can work out the ambiguity by looking ahead for the `:` as with collection expressions.
