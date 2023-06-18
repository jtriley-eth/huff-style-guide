# Huff Style Guide

_A mostly reasonable approach to Huff styling._

This style guide is based on common conventions with deviations where reasonable in the context of
huff. Optimization is around readability, control flow clarity, and in some cases, redundant
explanation of functionality.

## Table of Contents

- [Huff Style Guide](#huff-style-guide)
  - [Table of Contents](#table-of-contents)
  - [Basic Rules](#basic-rules)
  - [Macros and Functions](#macros-and-functions)
  - [Naming](#naming)
  - [Application Binary Interface](#application-binary-interface)
  - [Alignment](#alignment)
  - [Stack Comments](#stack-comments)
  - [Developer Documentation](#developer-documentation)

## Basic Rules

- One opcode per line with the following exceptions.
  - The first opcode of the line is followed by `dup1` instructions.
  - The first opcode is a pointer and the second is a respective "\*load" instruction.
  - A standard function dispatcher is used.
  - A return or revert with no returndata.

```
// ok
#define macro DISPATCHER() = takes (0) returns (0) {
    0x00 calldataload 0xe0 shr
    dup1 [SELECTOR_0] eq sel0 jumpi
    dup1 [SELECTOR_1] eq sel1 jumpi
    [SELECTOR_2] eq sel2 jumpi

    0x00 0x00 revert

    sel0:
        // ...

    sel1:
        // ...

    sel2:
        // ...
}

// ok
#define macro POINTER_GETTER() = takes (0) returns (0) {
    0x00 calldataload   // [selector_word]
    0x00                // [mem_ptr, selector_word]
    mstore              // []
    0x00 mload          // [selector_word]
}

// bad
#define macro MATHS() = takes (0) returns (0) {
    0x04 calldataload   // [arg0]
    0x02 dup1 add mul   // [???]
}
```

- Library files prefixed with `lib`.

```
#include "./libabi.huff";
#include "./libconst.huff";
#include "./liberr.huff";
```

- Items are ordered in each file as follows (disregarding comments).
  1. include directive
  2. application binary interface
  3. constant declaration
  4. `CONSTRUCTOR` macro
  5. `MAIN` macro
  6. macro declaration
  7. function declaration

```
#include "libstuff.huff";

#define function get() view returns (uint256)

#define constant SLOT = FREE_STORGE_POINTER()

#define macro CONSTRUCTOR() = takes (0) returns (0) {
    0x69    // [number]
    [SLOT]  // [slot, number]
    sstore  // []
}

#define macro MAIN() = takes (0) returns (0) {
    0x00 calldataload 0xe0 shr
    __FUNC_SIG(get) eq get_number jumpi
    0x00 0x00 revert

    get_number:
        GET_NUMBER()
}

#define macro GET_NUMBER() = takes (0) returns (0) {
    [SLOT]      // [slot]
    sload       // [number]
    0x00        // [mem_ptr, number]
    mstore      // []
    0x20        // [len]
    0x00        // [mem_ptr, len]
    return      // []
}
```

## Macros and Functions

- Always specify macro and function stack input and output counts.

```
// ok
#define macro ADD() = takes (2) returns (1) {
    // takes:   // [a, b]
    add         // [sum]
}

// bad
#define macro SUB() = takes (0) returns (0) {
    sub
}
```

- Template arguments go on a single line unless the total line length would be greater than 100 characters.
  - If multiple lines must be used, each tempalte argument goes on its own line.

```
// ok
#define macro THING(
    argument_number_1,
    argument_number_2,
    argument_number_3,
    argument_number_4,
    argument_number_5
) = takes (0) returns (0) {
    // ...
}

// bad
#define macro THING(argument_number_1, argument_number_2, argument_number_3, argument_number_4, argument_number_5) = takes (0) returns (0) {
    // ...
}

// really bad, go back to h*skell
#define macro THING(argument_number_1, argument_number_2, argument_number_3, argument_number_4,
                    argument_number_5) = takes (0) returns (0) {
    // ...
}
```

## Naming

- Storage slot constants are suffixed with `_SLOT`.

```
#define constant INITIAL_THING = 0x00
#define constant THING_SLOT = FREE_STORAGE_POINTER()
```

- Selector constants are suffixe with `_SEL`.

```
#define constant GET_SEL = 0x6d4ce63c
```

- Calldata and memory pointer constants are suffixed with `_PTR`.

```
#define constant ARG0_PTR = 0x04
#define constant FREE_MEM_PTR = 0x40
```

- Type cast macros and functions are named as `TO_TYP` where `TYP` is the respective type
  - `uint8` -> `TO_U8`
  - `int8` -> `TO_I8`
  - `bytes1` -> `TO_B1`
  - etc

```
#define macro TO_U8() = takes (1) returns (1) {
    // takes:   // [number]
    0xff        // [u8_mask, number]
    and         // [number_u8]
}
```

- Macros and function relating to the external ABI are to be screaming snake case.

```
#define function getByIndex(uint256) view returns (uint256)

#define macro GET_BY_INDEX() = takes (0) returns (0) {}
```

## Application Binary Interface

- Solidity-style ABI specification is declared in the final contract file and uses one of:
  - `__FUNC_SIG(fnName)`
  - `FN_NAME_SEL`

```
#define function get() view returns (uint256)
#define function set(uint256) nonpayable returns ()

#define macro MAIN() = takes (0) returns (0) {
    0x00 calldataload 0xe0 shr
    dup1 __FUNC_SIG(get) eq get_number jumpi
    __FUNC_SIG(set) eq set_number jumpi

    0x00 0x00 revert

    get_number:
        // ...

    set_number:
        // ...
}
```

- Bespoke ABI specifications are declared in a library, conventionally `libabi.huff`.
- Macros relating to dispatching are prefixed with `IS_` and are named in screaming snake case.
- Documentation around ABI encoding for each function are to be included here.
  - More details on documentation in the [developer documentation](#developer-documentation) section.

```
#define constant GET_SEL = 0x00000001
#define constant SET_SEL = 0x00000002

/// ## Is Get Call
///
/// ### Calldata Layout
///
/// `selector`
///
/// | name     | size (bytes) |
/// | -------- | ------------ |
/// | selector | 4            |
#define macro IS_GET() = takes (1) returns (1) {
    // takes:       // [selector]
    [GET_SEL]       // [get_selctor, selector]
    eq              // [is_get_selector]
}

/// Is Set Call
///
/// Calldata Layout
///
/// `selector . number`
///
/// | name     | size (bytes) |
/// | -------- | ------------ |
/// | selector | 4            |
/// | number   | 32           |
#define macro IS_SET() = takes (1) returns (1) {
    // takes:       // [selector]
    [SET_SEL]       // [set_selctor, selector]
    eq              // [is_set_selector]
}
```

## Alignment

- Top level declarations are prefixed with no spacing.
- Spaces are preferred over tabs.
- Space width for each level of indentation is four.
- Items inside of top level declarations are at one indentation level.
- Items following a conditional jump or jump destination add four more spaces of indentation.
  - Repeat this process recursively.

```
#define macro MAIN() = takes (0) returns (0) {
    0x04 calldataload           // [arg]
    is_truthy                   // [is_truthy_dest, arg]
    jumpi                       // []
        0x00 0x00 revert        // []

    is_truthy:
        0x24 calldataload       // [arg]
        is_also_truthy          // [is_also_truthy_dest, arg]
        jumpi                   // []
            0x00 0x00 revert    // []

        is_also_truthy:         // []
            0x00 0x00 return    // []
}
```

## Stack Comments

- Any nonzero stack input macro or function's first internal line contains a comment indicating stack state.
  - Prefix it with `// takes:` and align the initial stack state with `// [...]`.
  - A second `// ` prefixing the stack state is not syntactically required, but aligns the stack state with subsequent comments.
- Any nonzero stack output macro or function's last line contains the stack output implicitly, so no further lines are needed.

```
#define macro ADD() = takes (2) returns (1) {
    // takes:   // [a, b]
    add         // [sum]
}
```

- All stack comments in a macro or function must align to the same position.
  - The stack comment alignment is derived from a single tab spacing from the longest line.
  - Additional indentation prefixed to certain lines does not affect stack comment alignment.

```
#define macro MAIN() = takes (0) returns (0) {
    0x04 calldataload           // [arg]
    is_truthy                   // [is_truthy_dest, arg]
    jumpi                       // []
        0x00 0x00 revert        // []

    is_truthy:
        0x24 calldataload       // [arg]
        is_also_truthy          // [is_also_truthy_dest, arg]
        jumpi                   // []
            0x00 0x00 revert    // []

        is_also_truthy:         // []
            0x00 0x00 return    // []
}
```

## Developer Documentation

> Note: At the time of writing, `huffc` does not include any devdoc or similar for documentation,
> so this section is subject to change with changes to `huffc`.

- File leve devdoc is prefixed with double slash and exclamation mark (`//!`) and is at the top of the file before any code.
  - Contains the title and brief description of the file.

```
//! # Stuff Library
//!
//! This library does stuff.
```

- Item level devdoc is prefixed with triple slash (`///`) and exists directly above macro, constant, and function declarations.
  - Contains the item's:
    - Title.
    - "Directives" with ordered list of steps.
    - "Panics" with unordered list of panic cases.
    - "Template Arguments" with unodered list of template arguments.
    - Miscellaneous considerations

```
/// ## Computes the Signature Hash
///
/// ### Directives
///
/// 1. Copies calldata (without the signature array) to memory.
/// 2. Appends the chain id to the calldata in memory.
/// 3. Hashes the data.
/// 4. Writes the prefix "\x19Ethereum Signed Message:\n32" to memory
/// 5. Writes the intermediate hash to memory directly after the prefix
/// 6. Hashes the prefix and intermediate hash.
///
/// ### Pseudocode Representation
///
/// ```
/// hash("\x19Ethereum Signed Message:\n32", hash(id, target, value, deadline, payload))
/// ```
#define macro COMPUTE_HASH() = takes (1) returns (1) {
    // ...
}
```

- [Github flavored markdown](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) syntax is used for all devdocs.
  - Only a single H1 header may be used per file.
  - An empty line must precede and follow each header line.
  - Item titles are H2 headers.
  - Item devdoc subheaders start at the H3 header.

```
// -------------------------------------------------------------------------------------------------
//! # ECDSA Library
//!
//! Contains ECDSA Operations.

/// ## Computes the Signature Hash
///
/// ### Directives
///
/// 1. Copies calldata (without the signature array) to memory.
/// 2. Appends the chain id to the calldata in memory.
/// 3. Hashes the data.
/// 4. Writes the prefix "\x19Ethereum Signed Message:\n32" to memory
/// 5. Writes the intermediate hash to memory directly after the prefix
/// 6. Hashes the prefix and intermediate hash.
///
/// ### Pseudocode Representation
///
/// ```
/// hash("\x19Ethereum Signed Message:\n32", hash(id, target, value, deadline, payload))
/// ```
#define macro COMPUTE_HASH() = takes (1) returns (1) {
    // ...
}
```
