# List patterns for enumerables

Champion issue: <>

## Summary

Extends list patterns to be able to be used with enumerables that are not countable or indexable. `items.Where(...) is [var singleItem]`, or `is []`, or `is [p1, .., p2]`.

The pattern will be evaluated without multiple enumeration. The only slice pattern supported by this proposal is the discarding slice pattern, where `..` is not followed by another pattern referring to the slice.

## Motivation

## Detailed design

Any list pattern will be supported for an enumerable type (a type supported by `foreach`) if the same pattern would be supported by a type that is countable and indexable, but not sliceable. Thus, for the enumerable types gaining support through this proposal, it will be an error for a slice pattern to contain a subpattern. It is an existing requirement that the type additionally be sliceable in order for the slice pattern to have a subpattern; that requirement is not changing in this proposal.

No new syntax is involved in this proposal.

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
