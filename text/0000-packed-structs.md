- Feature Name: packed-structs
- Start Date: 2016-12-25
- RFC PR:
- Pony Issue:

# Summary

Add program annotations allowing programmers to declare packed structures, i.e. structures without implementation-specific padding between members.

# Motivation

In C, packed structures are a common idiom when doing I/O. Pony is currently unable to interface with this kind of structures, adding this capability would be good for the FFI and the language.

# Detailed design

The proposed annotation is `packed`, and would affect `struct` declarations:

```pony
struct \packed\ MyPackedStruct
  // Members.
```

The code generation for these structures will use LLVM's packed structures, which are very simple to use.

# How We Teach This

A paragraph will be added to the explanation of structures in the tutorial. In particular we will stress that, as normal Pony structures should only be used with normal C structures, packed Pony structures should only be used with packed C structures.

# How We Test This

The implementation will directly map onto LLVM's facilities. We'll assume that their implementation works.

# Alternatives

Not implementing this will leave the packed structures deficiency in the FFI unresolved.

# Unresolved questions

None.
