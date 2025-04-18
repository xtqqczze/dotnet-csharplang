# Readonly setter calls on non-variables

Champion issue: <https://github.com/dotnet/csharplang/issues/9174>

## Summary

Permits a readonly setter to be called on _all_ non-variable expressions:

```cs
var c = new C();
// Remove the current CS1612 error, because ArraySegment<T>.this is readonly:
c.ArraySegmentProp[10] = new object();

// Invocation expressions already omit the CS1612 error when the setter is readonly:
c.ArraySegmentMethod()[10] = new object();

// In limited cases, ref-returning indexers can be used to work around this:
c.RefReturningIndexerWorkaround[10] = new object();

class C
{
    public ArraySegment<object> ArraySegmentProp { get; set; }
    public ArraySegment<object> ArraySegmentMethod() => ArraySegmentProp;

    public Span<object> RefReturningIndexerWorkaround => ArraySegmentProp.AsSpan();
}
```

Currently, the code above gives the error CS1612 "Cannot modify the return value of 'C.ArraySegmentProp' because it is not a variable." This restriction is unnecessary when the setter is readonly. The restriction is there to remind you to assign the modified struct value back to the property. But there is no supported modification to the struct value when the setter is readonly, so there is no reason to assign back to the property.

```cs
// Requested by the current CS1612 error:
var temp = c.ArraySegmentProp;
temp[10] = new object();
c.ArraySegmentProp = temp; // But this line is purposeless; 'temp' cannot have changed.
```

## Motivation

Folks who want to simulate named indexers in C# use properties that return a wrapper with an indexer on it. The wrapper that holds the indexer must be a class today, which increases GC overhead. The only reason the wrapper cannot be a struct is because of the restriction which this proposal removes:

```cs
var c = new C();

// Proposal: no error because the indexer's set accessor is readonly.
c.SimulatedNamedIndexer[42] = new object();

class C
{
    public WrapperStruct SimulatedNamedIndexer => new(this);

    public readonly struct WrapperStruct(C c)
    {
        public object this[int index]
        {
            // Indexer accesses private state or calls private methods in 'C'
            get => ...;
            set => ...;
        }
    }
}
```

This addresses recurring community requests. These requests are often around document object models, entity component systems, numerics and geometry, and other modeling scenarios, but sometimes more ordinary helpers or utilities. This is the common thread: folks would like to not have to declare a pair of accessor methods when the accessors would make logical sense together as a single property or indexer. And, they don't want a class wrapper providing the API, they want a readonly struct wrapper.

```cs
// Undesirable:
c.SetXyz("key", c.GetXyz("key") + 1);

// Desirable:
c.Xyz["key"]++;
```

These use cases would no longer be blocked if CS1612 is fully updated with an understanding of readonly structs and readonly members.

## Detailed design

The CS1612 error is not produced for assignments where the setter is readonly. (If the whole struct is readonly, then the setter is also readonly.) The setter call is emitted the same way as any non-accessor readonly instance method call.

### Null-conditional assignment

The [Null-conditional assignment](https://github.com/dotnet/csharplang/blob/main/proposals/null-conditional-assignment.md) proposal enables the following syntax:

```cs
a?.b = value;
a?.b.c = value;
a?[index] = value;
```

CS1612 errors will be produced for this new syntax in the same way that they are for the existing syntax below:

```cs
a2.b = value;
a2.b.c = value;
a2[index] = value;
```

Thus, when the setter is readonly, the assignments will be permitted in both cases, and when the setter is not readonly, the normal CS1612 error for structs will remain in both cases.

### InlineArray

InlineArray properties are covered by a separate error which this proposal does not affect:

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

It's desirable for this error to remain, because the setter _does_ mutate the struct value.

## Specification

### Waiting for v8 and corrected v7 specification

The published v7 spec and the draft specs are all missing the wording that removed the CS1612 error for invocation expressions. This is tracked by <https://github.com/dotnet/csharpstandard/issues/1277> which suggests the following addition to describe the current behavior:

[§12.21.2](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/expressions.md#12212-simple-assignment) _Simple assignment_
> When a property or indexer declared in a _struct_type_ is the target of an assignment, **unless the _struct_type_ has the `readonly` modifier ([§16.2.2](https://github.com/dotnet/csharpstandard/blob/standard-v7/standard/structs.md#1622-struct-modifiers)) and the instance expression is an invocation**, the instance expression associated with the property or indexer access shall be classified as a variable. If the instance expression is classified as a value, a binding-time error occurs.

Then, the v8 spec is still in draft and the updates for readonly members have not yet merged. When it merges, it will need to redefine the condition as whether the setter itself is readonly, directly or indirectly.

### Spec updates

This proposal would remove the text "**and the instance expression is an invocation**" from the paragraph mentioned above in [Waiting for v8 and corrected v7 specification](#waiting-for-v8-and-corrected-v7-specification).

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

For the same reasons as above, this error is unnecessary when the properties being initialized have readonly `set`/`init` accessors. The error could be made more granular, placed on each property initializer which calls a _non-readonly_ setter.

Even in the limited cases where a ref-returning indexer is applicable, it does not help with CS1918, where it still unnecessarily broad:

```cs
// ❌ CS1918 Members of property 'C.StructWithRefReturningIndexer' of type 'Span<object>' cannot be assigned with an object
// initializer because it is of a value type
_ = new C { StructWithRefReturningIndexer = { [42] = new object() } };
//          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

class C
{
    public ArraySegment<object> ArraySegmentProp { get; set; }
    public Span<object> StructWithRefReturningIndexer => ArraySegmentProp.AsSpan();
}
```

## Downsides

If this proposal is taken, it becomes a source-breaking change to remove the `readonly` keyword from a struct or setter. Without the `readonly` keyword, the errors would then be relevant and would reappear.

Due to what looks like an unintentional change in the compiler, this source-breaking change is already in effect when the setter is called on an invocation expression:

```cs
// Removing 'readonly' from S1 causes a CS1612 error.
M().Prop = 1;

S1 M() => default;

public readonly struct S1
{
    public int Prop { get => 0; set { } }
}
```

```cs
// Removing 'readonly' from S2.Prop.set causes a CS1612 error.
M().Prop = 1;

S2 M() => default;

public struct S2
{
    public int Prop { get => 0; readonly set { } }
}
```
