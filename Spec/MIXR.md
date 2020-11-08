Manually Insert and Cross-Resolve
---------------------------------
A non-intrusive linker and symbol resolver.

## Overview
MIXR takes multiple TABs/DECs as input and produces a single TAB/DEC as output. It is designed to do the minimum work possible such that AMP can load the result. Its primary functions are grouping of input sections into output sections and resolving import relocations to cross table relocations. The first of these is dictated by an input file known as a TRAC, and the second by another known as a SLDR -- these are compiled to a lightweight binary format before use.

# Terse Regional Arrangement of Code
The following is an example TRAC:
```
# N.B.: Comments are an authorial liberty for clarity
# -- a real TRAC has no facility for comments

"code" vs 4 ( # name, type, alignment
    main/main # file/section
    lib/text
)

"heap" rw 8 (
    main/heap
    .fill x80 # provide 128 bytes of empty space (undefined value)
    lib/closure
    main/sync ( # sections can overlap to infinite depth, provided at most one at any point is concrete
        lib/sync # concurrent with the start of main/sync
        .skip x400 # skip to 1kB from start of main/sync (undefined fill value)
        second/sync
    )
)
```

A TRAC must provide all sections to satisfy every cross relocation within it -- MIXR will fail if it does not.

# Symbol Lookup Dependency Resolution
The following is an example SLDR:
```
# See the remark on comments in the TRAC example

# A SLDR must start with imports, which must specify symbols
* @ kern > panic, exit~return # the output imports module `kern` with symbols `panic` and `return` (latter aliased to `exit`)
* > printf # the output imports lone symbol `printf`

# Other than imports, information flow in all resolutions is right-to-left

* < main @ main # the output exports symbol `main` from `main`

main < bar~foo, baz @ lib # `main` sees symbols `foo` and `baz` from `lib`, `foo` visible as `bar`
main @ obj1, obj2~obj3 3 `main` sees module `obj1` unaliased, and `obj3` with alias `obj2`
obj1 < printf @ * # even when referenceing external symbols,
main < exit, panic @ kern # source object is required
```

A SLDR must either resolve or forward to the output all imports from all inputs, and may only use symbols actually exported from its inputs -- MIXR will fail if either of these conditions are not met. Since each TAB has a single import and export list, there is no need for a SLDR to know about sections.
