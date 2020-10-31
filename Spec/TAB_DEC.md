# Text-Addressable Binary/Deferred-Embeddable Code
A segmented binary format for straightforward linking and loading.

## Structure
A TAB file has the following structure, not including the file header:
* Metatable (t)
  - arch to:b (32), OS to:b (32), ABI to:b (32)
  - extern table to:b (32), import table to:b (32)
  - section table to:b (32), export table to:b (32)
  - extern table l:b (16), import table l:b (16)
  - section table l:b (16), export table l:b (16)
  * arch:a, OS:a, ABI:a
  * Module Table (m)
    - module name mo:b (16), ABI mo:b (16) [repeat]
    - 0 (16), bare symbol ABI mo:b (16) [repeat]
    * name:a [repeat], ABI:a [repeat]
  * Section Table (t)
    - name to:b (16), section to:b (32), l:b (32) [repeat]
    * name:a [repeat]
  * Import Table (i)
    - name io:b (16), module mi (14), module 0 (1), label/constant (1) [repeat]
    - name io:b (16), ABI mi (14), symbol 1 (1), label/constant (1) [repeat]
    * name:a [repeat]
  * Export Table (e)
    - name eo:b (16), section ti (13), parity (3), symbol si (14), parity (1), label/constant (1) [repeat]
    * name:a [repeat]
  * Section (r) [repeat]
    - code start ro:b (24), log2(alignment)-1 (5), type (3)
    - share table ro:b (32)
    - cross table ro:b (32)
    - relocation table ro:b (32)
    - code l:b (32)
    - share table l:b (16), cross table l:b (16), relocation table l:b (16)
    * Code (p)
    * Share Table (s)
      - po:b/value (32) [repeat]
    * Cross Table (x)
      - section ti (13), parity (2), label/constant (1), symbol si (16) [repeat]
    * Relocation Table
      - po:b (32), import ii (14), extern 0 (1), terminal/non-terminal (1), low:- (5), po:- (3), high:- (5), op (3) [repeat]
      - po:b (32), cross xi (14), intern 1 (1), terminal/non-terminal (1), low:- (5), po:- (3), high:- (5), op (3) [repeat]

**Legend:**
  - `Xo` = offset from start of `X`
  - `Xi` = index into table `X`
  - `l` = length
  - `:b` = in bytes
  - `:-`: bit/in bits
  - `:a` = as null-terminated ASCII string
  - `TYPE 0/1` = tag bit with value 0/1, indicating some other field is `TYPE`
  - `(XX)` = field is `XX` bits long
  - `[repeat]` = variable number of identically-structured fields
  - `parity` = any data -- may or may not actually be used for error checking

With this structure, the entrypoint to the file is a fixed-length, known-offset table, and every subsequent dereference is of known location and length. THus, accessing any part of the file is a constant-time operation. The counted length of any part of the above structure is the combined length of all of its sub-parts -- in particular, the length of a table includes the strings referenced by its fields. `*` parts are variable-length and may be reordered relative to each other, and `[repeat]` parts relative to themselves and adjacent `[repeat]` parts, as long as the locations and lengths of these parts are listed in the relevant places. With the exception of relocations, there are no ordering constraints on table rows.

### Metatable
The **metatable** contains the whole payload of the file. Its fixed-length portion contains the offsets and lengths of the major tables, as well as offsets of strings that identify the Zig-style target triple (architecture, OS, ABI) of the binary. Note that the entire TAB shares the same triple -- MIXR will refuse to run if asked to link objects of incompatible architecture or OS, or to expose symbols of multiple ABIs. (This does not prevent objects from using modules of other ABIs internally, or the user asserting that a particular architecture or OS is compatible with another.) The metatable does not contain its own offset or length, or state that the file is a TAB; that is strictly the business of RCRD. Sub-tables are assumed not to exceed 64kB, and TAB files themselves are assumed not to exceed 4GB -- I believe there are more intelligent ways to solve any problem that might push an executable file to these limits.

### Module Table
This contains the names of the **modules** (that is, other TABs) with which the TAB expects to be linked, as well as their ABIs. It can also contain ABIs with no associated module, for bare symbol imports. A sophisticated linker might decide not to include TABs with no imported symbols -- however, the base MIXR implementation does not do this, so a specific "none" module code is needed.

### Section Table
This is the real meat of the header: it contains the names and offsets of the program **sections**. Unlike most parts of the TAB, section offsets are given relative to the metatable, and sizes can be up to the whole TAB -- this is because sections might take up most of the file, there might be only one, and it would be simply impractical to include all of them in the section table. The section table is the only table in a TAB with non-power-of-two aligned fields, and believe me, it drives me insane.

### Import Table
This lists the external symbols which the TAB expects MIXR to provide, whether they be from a module (`module 0`) or alone (`symbol 1`), labels (`label 0`) or constants (`constant 1`). Even in the case of a lone symbol, an index into the module table is required to retrieve the ABI -- for compactness, there are no rules against reusing a module entry for this purpose.

### Export Table
This lists the internal symbols which MIXR may provide to other TABs, their native sections, and whether they are labels or constants. It may seem strange that the symbols themselves and their metadata are kept separate, even in the case of constants, which can't really have a "native" section -- this is for symmetry with the import table, and uniformity of bit lengths. Note that there is no mechanism for re-exporting imports -- if that is to be done, it will be done by MIXR.

### Sections
Each section lists attributes of its payload: its **alignment**, ranging from halfword to 32-bit address space in exponential steps; and its **type**, which may be *read-only*, *read-write*, *write-read*, *hidden code*, *visible code*, *input*, *output*, or *I/O*. It also has tables of its own: the **relocation table**, the **cross table**, and the **share table**.

**Share Table**
To save space, the top-level export table does not list program offsets. Instead, it indexes into a section's **share table**, which lists offsets of labels and values of constants. The same mechanism is used by the cross table (below). This indirection also allows for easier incremental linking, although incremental loading still requires more advanced techniques. Note that whether a symbol is a label or a constant is determined externally, and there is no explicit rule against interpreting any symbol as both -- as well as keeping table entries word-aligned, this also simplifies the implementation of GOT-based compilation strategies.

**Cross Table**
Some relocations (below) concern symbols in the same TAB, but a different section (as sections may be reordered, and hence their relative offsets not known at assemble/compile time). As these symbols may not be exported (and are certainly not imported), it is necessary to list their section and respective share table index. This is done in the **cross table**, which may then be indexed by the relocation table.

**Relocation Table**
This lists the **relocations** in the code, that is, information that cannot be completely filled in until link time as it references external data. These may be external symbols or symbols in other sections. MIXR performs relocation operations. Relocations are always listed in program order, although the order of their application is not defined. Multiple relocations may target the same offset. Unlike most paradigms, TAB relocations can never change the length of the program -- this enables linking and loading to be done in-place.

A relocation operation:
- finds the (high-low+1)-length bit slice at the specified program byte and bit offset (accounting for platform endianness)
  - if this relocation is non-terminal, collects consecutive relocations until the terminal, and checks that they have the same operation and unknown, and contiguous bit ranges -- fails linking if not
- constructs a left operand by sign-extending the known range and zeroing lower bits
- constructs a right operand by subtracting the program offset from a label unknown, or simply forwarding a constant unknown
- applies the specified operator, which may be: add, subtract, multiply, flooring divide, logical right shift, bitwise and, bitwise or, or bitwise xor, with the computed operands
- writes the [high:low] bits of the result back to the original bit slice or slices

A terminal/non-terminal bit is necessary to prevent internal rounding errors. This has the unfortunate consequence of not leaving any encoding space for a sign bit -- however, this is only significant for division of large unsigned numbers, which is rare. Label relocations have a subtlety, in that the byte offset of the relocation itself may not be the byte offset of the instruction that contains it -- in additive or subtractive cases, a corrective factor at the program location will cancel this; in others, well, you deserve what you get if you try to do that.

## Deferred-Embeddable Code
A **DEC** is a subclass of TAB with no modules or imports, and a single section with a null pointer in place of a name and no relocations or crosses. It is so named because AMP can link them into a TAB during loading without depending on MIXR, or even load them directly given an entrypoint. (No guarantees are made if libraries are executed in this manner.)
