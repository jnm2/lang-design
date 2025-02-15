# List patterns for enumerables

Champion issue: <>

## Summary

Extends list patterns to be able to be used with enumerables that are not countable or indexable. `items.Where(...) is [var singleItem]`, or `is []`, or `is [p1, .., p2]`.

The pattern will be evaluated without multiple enumeration. The only slice pattern supported by this proposal is the discarding slice pattern, where `..` is not followed by another pattern referring to the slice.

## Motivation

## Detailed design

Any list pattern will be supported for an enumerable type (a type supported by `foreach`) if the same pattern would be supported by a type that is countable and indexable, but not sliceable. Thus, for the enumerable types gaining support through this proposal, it will be an error for a slice pattern to contain a subpattern. It is an existing requirement that the type additionally be sliceable in order for the slice pattern to have a subpattern; that requirement is not changing in this proposal.

Async enumerables are not supported. So far in the language, consumption of async enumerables requires the `await` keyword, highlighting where execution is suspended.

No new syntax is involved in this proposal.

### Design rationale

Enumerables cannot be assumed to represent a realized collection. An enumerable may represent an in-memory collection or a generated sequence, but it may also represent a remote query or an iterator method. An enumerable may return different results on each enumeration, and it may have side effects. Multiple enumeration is considered both a performance smell and a correctness issue. The .NET SDK and other popular tools have versions of warnings for multiple enumeration of the same enumerable.

Because of this, any slice subpattern that matches against items within the slice  would have to buffer the sliced items in the general case (such as `..var slice`). It would not be able to expose a Skip/Take-style enumerable composed over the original enumerable, because any consumption of the resulting sliced enumerable would be a second enumeration in addition to the first enumeration performed by the pattern match.

This proposal does not enable slice subpatterns because of the near certainty of needing to buffer the entire enumerable into memory. The workaround for those who want it would be to take on the buffering explicitly in their code by matching against `enumerable.ToList()` or `enumerable.ToArray().AsSpan()` or similar, where slice subpatterns are already available for use. Buffering the entire enumerable into memory can be an expensive operation, and doing this operation silently during pattern matching could be a pitfall.

In [LDM 2022-10-19](https://github.com/dotnet/csharplang/blob/main/meetings/2022/LDM-2022-10-19.md#allowing-patterns-after-slices), there was interest in restricting slice subpatterns initially but pursuing them later. In light of the above rationale, this proposal recommends not pursuing them.

### Evaluation

The pattern will be evaluated using the enumerator. The enumerator will be obtained and disposed in the same manner as the `foreach` statement. The order of operations will be the following:

- `GetEnumerator()` is called.
- For each list pattern element if any, up to and not including the `..` if present:
  - `MoveNext()` is called. If it returns false, evaluation ends, and the list pattern is not matched.
  - `Current` is accessed no more than once, and the element pattern is matched against the value it returns. A temporary variable may be introduced to avoid calling `Current` more than once. It is preferable to skip the `Current` call for any patterns which can match without reading an input value, such as discards or redundant patterns. If the element pattern fails to match, evaluation ends and the list pattern is not matched.
- If the end of the pattern has been reached:
  - `MoveNext()` is called. Evaluation ends, and the list pattern is matched if `MoveNext()` returned `false` and is not matched if it returned `true`.
- Otherwise, there is a discarding slice pattern (`..`). If there are no more element patterns following the slice pattern, evaluation ends and the list pattern is matched.
- Otherwise, there are patterns to match at the end of the enumerable:
  - A buffer is allocated, such as an array or inline array at the discretion of the implementation, with a capacity equal to the number of patterns following the slice pattern.
  - An attempt is made to fill the buffer. For each pattern following the slice pattern:
    - `MoveNext()` is called. If it returns false, evaluation ends, and the list pattern is not matched.
    - `Current` is accessed and its value is stored in the first available unwritten position in the buffer.
  - Once the buffer has been filled, enumeration continues and the buffer is used as a circular buffer:
    - `MoveNext()` is called. If it returns `false`, enumeration is finished and evaluation moves to the final step of evaluating the remaining patterns.
    - If it returns `true`, `Current` is accessed and its value is stored in the buffer, overwriting the oldest entry still in the buffer. Enumeration continues from the previous step.
  - Each pattern following the slice pattern is matched against the buffer entries in order, so that the oldest buffer entry is matched with the first pattern that follows the slice pattern, and the newest buffer entry is matched with the last pattern in the list pattern. Evaluation ends. If any pattern fails to match, and the list pattern is not matched. Otherwise, the list pattern is matched.
- The enumerator is disposed, if applicable. This step is not skipped when evaluation ends.

If the patterns following the slice pattern consist only of patterns which can match without reading an input value, such as discards or redundant patterns, then an implementation may omit the buffer and the `Current` calls. Rather than enumerating all remaining items, enumeration is only done once for each pattern following the slice pattern. For each, `MoveNext()` is called. If it returns `false`, evaluation ends and the list pattern is not matched. If it returns `true` once for each remaining pattern, evaluation ends and the list pattern is matched.

## Answered questions

### Allowing slicing on `IEnumerable` types

On `IEnumerable` types, should we allow `..var x` where `x` is an `IEnumerable` that the compiler implements, similar to collection expressions targeting `IEnumerable<T>`?

#### Answer

No, we think it will be a better experience if we only allow sub-patterns under a slice if the type is actually sliceable. ([LDM 2021-04-12](https://github.com/dotnet/csharplang/blob/main/meetings/2021/LDM-2021-04-12.md#list-patterns))

## Open questions

### Optimizing statically countable enumerables

Should the compiler be allowed to optimize patterns such as `[1, _, _]` by enumerating only one item, and assuming that `Length`/`Count` is well-behaved and can be checked rather than enumerating for the rest of the pattern?

### Optimizing runtime-countable enumerables

Similar to the previous question, should the compiler be allowed to use `TryGetNonEnumeratedCount` to avoid full enumeration for patterns such as `[1, _, _]`?
