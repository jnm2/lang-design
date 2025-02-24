# Readonly setter calls on non-variables

Champion issue: <>

## Summary

Permits a readonly setter to be called on a non-variable:

```cs
var c = new C();
// No error, because ArraySegment<T>.this is readonly
c.ArraySegmentProp[10] = new object();

class C
{
    public ArraySegment<object> ArraySegmentProp { get; set; }
}
```

Currently, the code above gives the error CS1612 "Cannot modify the return value of 'C.ArraySegmentProp' because it is not a variable." This restriction is meaningless when the setter is readonly. The restriction is there to remind you to assign the modified struct value back to the property. But there is no modification to the struct value, so there is no reason to assign back to the property.

```cs
// Requested by the current CS1612 error:
var temp = c.ArraySegmentProp;
temp[10] = new object();
c.ArraySegmentProp = temp; // But this line is purposeless; 'temp' cannot have changed.
```

## Motivation

TODO: Enables named indexers to be simulated without GC overhead
TODO: Has been lifted for method returns?

## Detailed design

The CS1612 error is not produced for assignments where the setter is readonly. (If the whole struct is readonly, then the setter is also readonly.) The setter call is emitted the same way as any non-accessor readonly instance method call.

### Notes

InlineArray properties are covered by a separate error which this proposal does not effect:

```cs
var c = new C();

// ❌ CS0313 The left-hand side of an assignment must be a variable, property or indexer
c.InlineArrayProp[42] = new object();

class C
{
    public InlineArray43<object> InlineArrayProp { get; set; }
}

[System.Runtime.CompilerServices.InlineArray(43)]
public struct InlineArray43<T> { public T Element0; }
```

It's desirable for this error to remain, because the setter does mutate the struct value.

## Specification

TODO

## Expansions

There's another location where this kind of assignment is blocked, which is in object initializers:

```cs
// ❌ CS1918 Members of property 'C.ArraySegmentProp' of type 'ArraySegment<object>' cannot be assigned with an object
// initializer because it is of a value type
_ = new C { ArraySegmentProp = { [42] = new object() } };
//          ~~~~~~~~~~~~~~~~

class C
{
    public ArraySegment<object> ArraySegmentProp { get; set; }
}
```

This error is inappropriate when the properties being initialized are readonly. It could be made more granular, placing the error on each property initializer which calls a _non-readonly_ setter.
