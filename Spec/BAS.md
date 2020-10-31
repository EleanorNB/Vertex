# Bootstrap Assembly Syntax
An assembler and language schema designed for self-hosting.

## Overview
Most constructs are preceded by sigils, and any lists have explicit separators or groupings. This is so that the assembler can be implemented as a finite state machine with one-character pass-ahead. Spaces are insignificant except for symbol separation; newlines terminate constructs unless preceded by a separator or inside a delimited block.

Assembly is done per source file, and outputs a single TAB or DEC (TABs have sections and imports, DECs do not). External labels and TABs/DECs are coordinated by MIXR.

## Directives and Statements
Directives start with `.` and consist of alphanumerics. Directives may have arguments, and those arguments may themselves have arguments: top-level arguments are separated with `;`, the first of which is optional; sub-arguments are separated with `,`, likewise. Directive statements run until the first newline after the directive, an argument, or a sub-argument not terminated with `;` or `,`; `.` on its own can be used to terminate multiline statements without adding an argument.

### Example
```
.triple;
    arch "arm64v8.6+neon+tme";
    os "vertexbeta";
    abi "eabivfp";
.

.section "code"; type vs; align $4

# This particular directive is a no-op in base, hence why it is allowed outside leading position
# -- it provides upward compatibility for more sophisticated linkers
:myfunc .fn (
    # ...
)

.const x, $x45

:mydata
    .b $x00; $x11; $x22; $x33
```

## Sections and Attributes
A BAS file may contain multiple `.section` statements and assemble a TAB, or a single `.attr` statement and assemble a DEC. An `.attr` statement declares a type and an alignment; a `.section` statement also declares a name, which may be any string. Alignment is always a power of 2, at least 2 bytes and up to 4 gigabytes. Type depends on whether a file's data is **r**eadable or not, **w**ritable or not, e**x**ecutable or not, and whether AMP should copy it, **a**llocate it, or pass a **v**olatile mapping:
* `ro`, r---: Read-only. Constant data.
* `rw`, rw--: Read-write. Mutable data.
* `wr`, rw-a: Write-read. Uninitialised data.
* `hd`, --x-: Hidden code. Execute-only.
* `vs`, r-x-: Visible code. Read-execute.
* `ip`, r--v: Input. Read-only interface.
* `op`, -w-v: Output. Write-only interface.
* `io`, rw-v: I/O. Bidirectional interface.
Instructions and data directives are not permitted inside allocated or volatile sections. Type does not prevent a process from requesting additional memory or changing the permissions of its own data, provided it has the necessary capabilities.

## External Objects and Imports
External objects may be referenced and specific symbols imported with the `.extern` directive:
```
.extern "eabi";
    "termios" :clear, !t_w~"width", !t_h~"height"; # Can import labels or constants,
    "stdio" :printf, !p_len~"max_length", !c_w~"utf_mode" # with optional aliases
```
Multiple such statements are allowed per file, although object repeats are not allowed. In cases where specific objects should be handled by MIXR, there is also `.import`: `.import; :foo; !bar "baz"` (the first semicolon is necessary to elide the ABI -- it defaults to the ABI of the file). To be used in this way, a symbol must first be `.export`ed: `.export :foo; !baz "bar"`. Note the lack of aliasing tilde in these cases -- it is not necessary, as the aliases fit in standard argument position.

## Operators and Instructions
Operators are the only unadorned leading elements. Arguments after the first are separated with commas. Instructions run until the first newline not preceded by a comma on a non-empty line. Modifiers follow the operator, joined with `.`; predications always come last, joined with `|`. Instructions may be separated on the same line with `;`. Registers are unadorned except for their initial letters.

### Example
```
    ldr.b r0, [r1], $1
    add r0, r0, r3; cmp r0, $0
    str.b|lt r0,
             [r2, #1]!; ret
```

## Symbols
There are five kinds of symbols: **absolute**, **relative**, **constant**, **external**, and **literal**. The first four must consist solely of alphanumerics and underscores. There is no first character requirement, however users may implement their own naming conventions.
* Absolute labels refer to locations in the output TAB or DEC. They are declared at the beginning of a line as `:label`, and referenced from anywhere in the file in the same format. There may not be duplicates.
* Relative labels are declared at the beginning of a line as `%label`. They differ from absolute labels in allowing duplicates, as well as requiring a direction when referencing (`%>label` for look-ahead, `%<label` for look-behind, `%.` for the current instruction). Referenced relative labels must be within the same section.
* Constants are declared as `.const x $4; y $5`, and referenced as `!x`. Labels of any kind and later-defined constants are not permitted in constant declarations.
* External symbols are declared in other files as either constants or absolute labels, and then `.export`ed. They must then be imported with either `.extern` or `.import`. External labels are referenced as `@label`, external constants are referenced as `^x`.
* Numeric literals begin with `$`, followed by an optional sign ([`+`]/`-`), then an optional base (`x`/`b`/[`d`]), and finally the digits. Their range is determined by their context -- in particular, a sign is disallowed in unsigned contexts, and mandatory in signed contexts. Exceeding this range is a fatal error. Constants have a (platform width + 1)-bit signed range.
Constants and literals are collectively known as *numerals*.

Note that BAS is optimised for position-independent code -- regardless of whether a label is absolute or relative, it is always represented in the output as a relative offset. If the absolute address of a label is needed, it may be expressed as `%.+:label`.

Single binary operations are permitted in some circumstances: a label may have a positive or negative offset of any symbol type; a numeral may be sliced with `[y-x]`, where `y` and `x` are the inclusive top and bottom of the bit range, as unadorned unsigned integers; and a literal or internal constant may be acted on by a numeral with one of the following operators:
* `+` (addition)
* `-` (subtraction)
* `*` (multiplication)
* `/` (flooring division)
* `<` (left shift, logical)
* `>` (right shift, logical)
* `|>` (right shift, arithmetic)
* `&` (bitwise and)
* `|` (bitwise or)
* `^` (bitwise xor)

(Note: in the output TAB, left shift is actually multiplication, and arithmetic right shift is actually flooring division -- their functionality is equivalent, and performance does not matter as much in this context. A sophisticated linker may apply this optimisation to improve its own efficiency, however this is not required.)

## Macros
For upward compatibility with more sophisticated assemblers, BAS defines a standard macro escape mechanism: enclosing macro code in ``. The syntax and semantics of code so enclosed is considered out of scope, and the base implementation will error if it detects such a sequence.

## Comments
Conmments begin with `#` and run to the end of the line.
