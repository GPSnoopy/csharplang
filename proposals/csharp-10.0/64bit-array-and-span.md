# Expand arrays, strings and spans to native-int lengths

## Summary

Expand arrays (e.g. `float[]`), `string`, `Span<T>` and `ReadOnlySpan<T>` to have 64-bit lengths on 64-bit platforms (also known as native lengths).

## Motivation

Currently in C#, arrays (and the other aforementioned types) are limited to storing and addressing up to 2^31 (roughly 2 billion) elements. This limits the amount of data that can be contiguously stored in memory and forces the user to spend time and effort in implementing custom non-standard alternatives like native memory allocation or jagged arrays. Other languages such as C++ or Python do not suffer from this limitation.

With every passing years, due to the ever-continuing increases in RAM capacity, this limitation is becoming evident to an ever-increasing number of users. For example, machine learning frameworks often deal with large amount of data and dispatch their implementation to native libraries that expect a contiguous memory layout.

At the type of writing this (2021-03-23), the latest high-end smartphones have 8GB-12GB of RAM while the latest desktop CPUs can have up to 128GB of RAM. Yet when using `float[]`, C# can only allocate up to 8GB of contiguous memory.

## Proposal

The proposed solution is to change the signature of arrays, `string`, `Span<T>` and `ReadOnlySpan<T>` to have an `nint Length` property and an `nint` indexing operator (similar to what C++ does with `ssize_t`), respectively superseding and replacing the `int Length` and `int` indexing operator.

As this would break existing C# application compilation, an opt-in mechanism similar to what is done with C# 8.0 nullables is proposed. By default, assemblies are compiled with *legacy lengths* but new assemblies (or projects that have been ported the new native lengths) can opt-in to be compiled with *native lengths* support.

## Future Improvements

This proposal limits itself to arrays, `string`, `Span<T>` and `ReadOnlySpan<T>`. The next logical step is to tackle all the standard containers such as `List<T>`, `Dictionary<T>`, etc.

## Language Impacts

When an assembly is compiled with *native lengths*, the change of type for lengths and indexing from `int` to `nint` means that existing code cannot compile out of the box and will need to be ported to properly support this new feature. Specifically the following constructs will need updating.

```c#
for (int i = 0; i < array.Length; ++i) { /* ... */ } // If length is greather than int, this will loop forever.

for (var i = 0; i < array.Length; ++i) { /* ... */ } // Same as above. A lot of code is written like this rather than using an explicit int type.
```

The correct version should be the following.

```c#
for (nint i = 0; i < array.Length; ++i) { /* .... */ }
```

**TODO Review existing casting C# rules with int and nint, and whether the C# compiler would allow int i to be compared to nint Length without a much needed error**


## Runtime Impacts

In order to accomodate these changes, various parts of the dotnet 64-bit runtime need to be modified. Using C# `nint` and C++ `ssize_t`, these modifications are in theory backward compatible with the existing 32-bit runtime.

The proposal assumes that these are always enabled, irrespective of whether any assembly is marked as *legacy lengths* or *native lengths*.

- The C++ runtime implementation of .NET arrays and strings need to used `ssize_t` to store their length (**TODO I believe this is already the case, verify it**).
- The .NET runtime implementation of `Span<T>` and `ReadOnlySpan<T>` need to internally store their length as `nint`.
- The .NET runtime implementation of `System.Array` needs to be updated to reflect the same type changes.
- The JIT needs to be aware of *native lengths* when generating code that accesses arrays and strings, for both the `Length` property and the indexing operator.
- The C#/JIT (**TODO which level implements foreach?**) needs to generate `foreach` code that is `nint` aware for arrays, `string`, `Span<T>` and `ReadOnlySpan<T>`.
- The GC implementation currently assumes that containers and arrays have up to 2^31 elements and outgoing references. This limitation needs to be lifted.

## Boundaries Between Assemblies

### Calling *Legacy Lengths* Assembly From *Native Lengths* Assembly

Without any further change, a *native length* assembly passing an array (or any other types covered by this proposal) with a large length (i.e. greater than 2^31) to a *legacy lengths* assembly would result into an undefined behaviour. The most likely outcoming would be the JIT truncating the `Length` property to its lowest 32-bit, causing the callee assembly to only loop on a subset of the given array.

The proposed solution is to follow C# 8.0 nullables guard and warn/error when such a boundary is crossed in an incompatible way. As with nullables, the user can override this warning/error (**TODO exact syntax TBD, can we reuse nullables exclamation mark?**).

**TODO should we also provide a utility method for safely checking at runtime a *native lengths* call into a *legacy lengths* assembly?**

In practice, applications rarely need to pass large amount of data to their entire source code and dependencies domain. We forsee that most applications will only need to make a small part of the source code and dependencies *native lengths* compatible. Thus enabling a smoother transition to a *native lengths*-only C# future version.

### Calling *Native Lengths* Assembly From *Legacy Lengths* Assembly

The proposed aforementioned runtime changes mean this should work as expected.

## FAQ

...


