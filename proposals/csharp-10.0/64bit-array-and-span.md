# Expand arrays, strings and spans to native-int lengths

## Summary

Expand arrays (e.g. `float[]`), `string`, `Memory<T>`, `ReadOnlyMemory<T>`, `Span<T>` and `ReadOnlySpan<T>` to have 64-bit lengths on 64-bit platforms (also known as native lengths).

## Motivation

Currently in C#, arrays (and the other aforementioned types) are limited to storing and addressing up to 2^31 (roughly 2 billion) elements. This limits the amount of data that can be contiguously stored in memory and forces the user to spend time and effort in implementing custom non-standard alternatives like native memory allocation or jagged arrays. Other languages such as C++ or Python do not suffer from this limitation.

With every passing years, due to the ever-continuing increases in RAM capacity, this limitation is becoming evident to an ever-increasing number of users. For example, machine learning frameworks often deal with large amount of data and dispatch their implementation to native libraries that expect a contiguous memory layout.

At the type of writing this (2021-03-23), the latest high-end smartphones have 8GB-12GB of RAM while the latest desktop CPUs can have up to 128GB of RAM. Yet when using `float[]`, C# can only allocate up to 8GB of contiguous memory.

## Proposal

The proposed solution is to change the signature of arrays, `string`, `Memory<T>`, `ReadOnlyMemory<T>`, `Span<T>` and `ReadOnlySpan<T>` to have an `nint Length` property and an `nint` indexing operator (similar to what C++ does with `ssize_t`), respectively superseding and replacing the `int Length` and `int` indexing operator.

As this would break existing C# application compilation, an opt-in mechanism similar to what is done with C# 8.0 nullables is proposed. By default, assemblies are compiled with *legacy lengths* but new assemblies (or projects that have been ported the new native lengths) can opt-in to be compiled with *native lengths* support.

## Future Improvements

This proposal limits itself to arrays, `string`, `Memory<T>`, `ReadOnlyMemory<T>`, `Span<T>` and `ReadOnlySpan<T>`. The next logical step is to tackle all the standard containers such as `List<T>`, `Dictionary<T>`, etc.

## Language Impacts

When an assembly is compiled with *native lengths*, the change of type for lengths and indexing from `int` to `nint` means that existing code will need to be ported to properly support this new feature. Specifically the following constructs will need updating.

```c#
for (int i = 0; i < array.Length; ++i) { /* ... */ } // If length is greather than what's representable with int, this will loop forever.

for (var i = 0; i < array.Length; ++i) { /* ... */ } // Same as above. A lot of code is written like this rather than using an explicit int type.
```

The correct version should be the following.

```c#
for (nint i = 0; i < array.Length; ++i) { /* .... */ }
```

Unfortunately, due to implicit conversion the C# 9.0 compiler does not complain when comparing an `int` to an `nint`. It is therefore proposed that the compiler warns when an implicit conversion occurs from `int` to `nint` for the following operations: `a < b`, `a <= b`, `a > b`, `a >= b` and `a != b`.

## Runtime Impacts

In order to accommodate these changes, various parts of the dotnet 64-bit runtime need to be modified. Using C# `nint` and C++ `ssize_t`, these modifications are in theory backward compatible with the existing 32-bit runtime.

The proposal assumes that these are always enabled, irrespective of whether any assembly is marked as *legacy lengths* or *native lengths*.

src/coreclr/vm/object.h

- The C++ runtime implementation of .NET arrays need to use `ssize_t` to store their length (`ArrayBase` in dotnet/runtime/src/coreclr/vm/object.h already has extra padding when compiling for 64-bit).
- The C++ runtime implementation of .NET strings need to use `ssize_t` to store their length (`StringObject` in dotnet/runtime/src/coreclr/vm/object.h does not have enough padding, this adds 4 bytes per string instancfe in 64-bit).
- The .NET runtime implementation of `Memory<T>`, `ReadOnlyMemory<T>`, `Span<T>` and `ReadOnlySpan<T>` need to internally store their length as `nint`.
- The .NET runtime implementation of `System.Array` needs to be updated to reflect the same type changes.
- The .NET standard libraries needs to be ported to *native lengths* (e.g. loops using `nint`, length arithmetic using `nint`, etc - `System.Linq` is a good example).
- The JIT needs to be aware of *native lengths* when generating code that accesses arrays and strings, for both the `Length` property and the indexing operator.
- The C# compiler and JIT compiler need to generate `foreach` code that is `nint` aware for arrays, `string`, `Span<T>` and `ReadOnlySpan<T>`.
- The GC implementation currently assumes that containers and arrays have up to 2^31 elements and outgoing references. This limitation needs to be lifted.

## External Tools Impacts

External tools such as debuggers (e.g. SoS) may have hard-coded assumptions about the memory layout of arrays and strings and will need to be updated to be compatible with the new runtime.

## Boundaries Between Assemblies

### Calling *Legacy Lengths* Assembly From *Native Lengths* Assembly

Without any further change, a *native length* assembly passing an array (or any other types covered by this proposal) with a large length (i.e. greater than 2^31) to a *legacy lengths* assembly would result into an undefined behaviour. The most likely outcome would be the JIT truncating the `Length` property to its lowest 32-bit, causing the callee assembly to only loop on a subset of the given array.

The proposed solution is to follow C# 8.0 nullables compiler guards and warn/error when such a boundary is crossed in an incompatible way. As with nullables, the user can override this warning/error (**TODO exact syntax TBD, can we reuse nullables exclamation mark? Or do we stick to nullable-like preprocessor directives? IMHO the latter is probably enough**).

In practice, applications rarely need to pass large amount of data to their entire source code and dependencies domain. We foresee that most applications will only need to make a small part of the source code and dependencies *native lengths* compatible. Thus enabling a smoother transition to a *native lengths*-only C# future version.

### Calling *Native Lengths* Assembly From *Legacy Lengths* Assembly

The proposed aforementioned runtime changes mean this should work as expected when passing a *legacy lengths* array as a function argument to a *native length* assembly.

However, a *native lengths* assembly method returning an array or string to a *legacy lengths* assembly is problematic. For example:

```c#
byte[] bytes = GetSomeBytes();
byte[] moreBytes = GetMoreBytes();
byte[] concat = bytes.Concat(moreBytes).ToArray(); // System.Linq
```

In this case the user cannot be relied upon to tell us whether its safe to do so, as the caller is in a *legacy lengths* assembly. It is therefore proposed that the JIT compiler adds an automatic runtime length check whenever a method returns an array, a `string`, a `Memory<T>` or a `ReadOnlyMemory<T>` that crosses the boundary from a *native lengths* assembly to a *legacy lengths* assembly. This applies to return values, reference arguments and output arguments. The check throws either `OutOfMemoryException` or `OverflowException` - **To Be Decided which one**.

**TODO Do we think this is an acceptable overhead? -Probably virtually inexistent for most applications- Do we have a better idea instead? For example, System.LINQ is extensively used, so we need to be careful**

**TODO Address comment from Levi Broderic (https://github.com/dotnet/runtime/issues/12221#issuecomment-805978753)**

*This would not be sufficient. The problem is indirection, such as through interfaces and delegate dispatch. It's not always possible to know what behavioral qualities the caller / target of an indirection possesses. (It's one of the reasons .NET Framework's CAS had so many holes in it. Indirection was a favorite way of bypassing the system.)*

*In order to account for this, you'd need to insert the check at entry to every single legacy method, and on the return back to any legacy caller. It cannot be done only at legacy / native boundaries.*

**TODO can we at least only do the check at boundaries when the method call is non-virtual?**

## FAQ

**Expanding strings to support 64-bit lengths add 4 bytes per instance. Do we accept this cost?**

.NET reference objects in 64-bit are already much larger than their 32-bit counterpart. Each reference pointer is 8 bytes instead of 4 bytes, and the object themselves have a minimum size of 24 bytes instead of 12 bytes (object header, method table pointer, and min size of instance field - all 8 bytes instead of 4 bytes).

Thus a string instance is currently at minimum 32 bytes, not including the actual string content. Adding an extra 4 bytes for the 64-bit length would likely push this to 40 bytes due to alignment.

This proposal assumes that such an overhead would be virtually unoticeable to most applications.


