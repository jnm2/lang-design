# Target-typed static member lookup

Champion issue: <TODO>

## Summary

This feature enables a type name to be omitted when it is the same as the target type.

It paves the way for discriminated unions to benefit from expressing existing construction and consumption concepts, while also making consumption nicer today for manual type-based discriminated unions.

Examples:

```cs
type.GetMethod("Name", .Public | .Instance | .DeclaredOnly);

control.ForeColor = .Red;
entity.InvoiceDate = .Now;
ReadJsonDocument(.Parse(stream));

// Production (static members on Option<int>)
Option<int> option = condition ? .None : .Some(42);

// Consumption (nested derived types)
return option switch
{
    .Some(var val) => val,
    .None => defaultVal,
};
```

## Motivation

LDM prior interest <https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-09-26.md#discriminated-unions>

DUs, existing nested type checks, BindingFlags

## Detailed design

In a target-typed location, there is a new primary expression which starts with `.` and is followed by an identifier.

### Target typing through overloadable operators

`~.None` should work.

Non-bitwise operators: `new(1) + new(2)`

### Misc

Same as with target-typed `new`, targeting a nullable value type should access members on the inner value type:

```cs
Point? p = new();
Point? p = .Empty;
```

Same as with target-typed `new`, overload resolution is not influenced by the presence of a target-typed static member expression. If overload resolution was influenced, it would become a breaking change to add any new static member to a type.

```cs
M(.Empty); // Overload ambiguity error

void M(string p) { }
void M(object p) { }
```

## Specification

`identifier type_argument_list?` is consolidated into a standalone syntax, `member_binding`, and this new syntax is then added as a production of the [§12.8.7 Member access](https://github.com/dotnet/csharpstandard/blob/draft-v8/standard/expressions.md#1287-member-access) grammar:

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

Expand target-typing through:

- overloadable operators: `.Public | .Instance`
- invocation expressions: `.Create(...)`

Compare to how target-typing works for `??`.

## Drawbacks

There are a couple of ambiguities, with [parenthesized expressions](#ambiguity-with-parenthesized-expression) and [conditional expressions](#ambiguity-with-conditional-expression).

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

To match the production and consumption sides even better, constructors may

## Alternatives

### Doing nothing

### No sigil

`GetMethod("Name", Public | Static)`

1. When target-typed names are added into the universe of available names to look up from, it's almost like tearing a wormhole in spacetime. It's a powerful event. The lookup rules change behavior. That's a good match for new syntax indicating "the rules are different here." You are traversing a wormhole to a different lookup universe. This is making the feature stand out, but not for the sake of standing out as a new feature. The places where the rules behave differently should be coupled with a visible, though still minimal, marker, or essential context is missing. If no such marker is in place, it will slow down understanding of code. Every identifier will need to be considered as to whether it is in a target-typing location and could be referring to something on that type. The presence of `.` makes reading much more efficient.

1. To avoid changes in meaning, this would have to prefer binding to other things in the current scope name, with target-typing as a fallback. This would result in unpleasant interruptions with no recourse other than typing out the full type name. These interruptions are expected to be often enough to hamper the success of the feature.

## Open questions

### Ambiguity with parenthesized expression

This is valid grammar today, which fails in binding if `A` is a type and not a value. The new grammar we're adding would allow this to be parsed as a cast followed by a target-typed static member lookup. This new interpretation is consistent with `(A)new()` and `(A)default` working today, but it is not in the least useful. `A.B` is a simpler and clearer way to write the same thing.

Should `(A).B` continue to fail, or be made to work the same as `A.B` when `A` is a type?

### Ambiguity with conditional expression

There is an ambiguity if target-typed static member lookup is used as the first branch of a conditional expression, where it would parse today as a null-safe dereference: `expr ? .Name : ...`

We can follow the approach already taken for the similar ambiguity in collection expressions with `expr ? [` possibly being an indexer and possibly being a collection expression.

Alternatively, target-typed static member lookup could be always disallowed within the first branch of a conditional expression unless surrounded by parens: `expr ? (.Name) : ...`. The downside is that this puts a usability burden onto users, since the compiler can work out the ambiguity by looking ahead for the `:` as with collection expressions.
