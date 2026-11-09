# BFSL - Language Design

## Contents

- [1. Language overview](#1-language-overview)
- [2. Example specification](#2-example-specification)
- [3. Declarations and types](#3-declarations-and-types)
  - [3.1. Top-level declarations and constants](#31-top-level-declarations-and-constants)
  - [3.2. Primitive types and arrays](#32-primitive-types-and-arrays)
  - [3.3. Structures](#33-structures)
  - [3.4. Enumerations](#34-enumerations)
  - [3.5. Unions and variant selection](#35-unions-and-variant-selection)
  - [3.6. Bitfield types](#36-bitfield-types)
- [4. Field and layout constructs](#4-field-and-layout-constructs)
  - [4.1. Endianness](#41-endianness)
  - [4.2. Field-level constraints](#42-field-level-constraints)
  - [4.3. Conditional fields](#43-conditional-fields)
  - [4.4. Remaining bytes](#44-remaining-bytes)
  - [4.5. Padding and layout](#45-padding-and-layout)
  - [4.6. Repeated elements](#46-repeated-elements)
  - [4.7. Size-bounded fields](#47-size-bounded-fields)
- [5. Validation and checksums](#5-validation-and-checksums)
  - [5.1. Validation blocks](#51-validation-blocks)
  - [5.2. Compile-time assertions](#52-compile-time-assertions)
  - [5.3. Checksum fields](#53-checksum-fields)
- [6. Expressions and built-ins](#6-expressions-and-built-ins)
- [7. Semantic analysis and evaluation](#7-semantic-analysis-and-evaluation)
  - [7.1. Evaluation order and errors](#71-evaluation-order-and-errors)
  - [7.2. Input region and trailing bytes](#72-input-region-and-trailing-bytes)
- [8. Field modifier compatibility](#8-field-modifier-compatibility)
- [9. Intermediate Representation](#9-intermediate-representation)
- [10. UML frontend and UML Profile](#10-uml-frontend-and-uml-profile)
- [Appendix A. Reference tables](#appendix-a-reference-tables)
  - [A.1. Keywords and names](#a1-keywords-and-names)
  - [A.2. Types](#a2-types)
  - [A.3. Built-in functions and values](#a3-built-in-functions-and-values)
  - [A.4. Operators](#a4-operators)
  - [A.5. Checksum algorithms](#a5-checksum-algorithms)
  - [A.6. Literals, comments, and names](#a6-literals-comments-and-names)


## 1. Language overview

BFSL (Binary Format Specification Language) is a declarative language for describing binary data formats. A specification defines the structure of binary data, the interpretation of individual fields, layout rules, and validation constraints. The intention is to describe *what* a binary format looks like rather than implement a parser procedurally.

The language provides six kinds of top-level declarations:

- `struct` - defines a composite data structure consisting of ordered fields. 
- `union` - defines alternative interpretations of the same data region. A `switch` field selects the active alternative.
- `enum` - defines named integer values with an explicitly specified underlying type; an enum can be closed or open.
- `bitfield` - defines a fixed-width unsigned storage unit divided into named bit fields.
- `format` - defines a top-level binary format. 
- `const` - defines a reusable compile-time value.

## 2. Example specification

The following example illustrates the proposed syntax using a simplified network packet format. It is an illustrative example rather than a complete protocol specification.

```rust
const MAX_PAYLOAD_SIZE: u16 = 4096;
const PROTOCOL_VERSION: u8 = 1;

enum MessageKind: u8 {
    DATA = 1;
    ACK  = 2;
    SYN  = 3;
}

bitfield PacketFlags: u8 {
    bit_order msb_first;

    sequence_present: bits[1];
    extended_mode: bits[1];
    reserved: bits[6] expect 0;
}

struct CommonHeader {
    version: u8 expect PROTOCOL_VERSION;
    kind: MessageKind;
    reserved: u8 where $value == 0;

    assert size_of(Self) == 3;
}

format DataPacket {
    endian big;

    magic: bytes[2] expect hex("BF 01");
    header: CommonHeader;

    flags: PacketFlags;
    payload_length: u16 where $value > 0 && $value <= MAX_PAYLOAD_SIZE;

    if flags.sequence_present != 0 {
        sequence_number: u32;
    }

    payload: bytes[payload_length];

    checksum: u32 using crc32 {
        covers = [header, flags, payload_length, sequence_number, payload];
    }

    assert offset_of(payload_length) == 6;

    validate {
        (flags.extended_mode == 0) || payload_length >= 4;
    }
}
```

The `sequence_number` field is conditional. If it is absent, it contributes no bytes to the checksum input (see [Section 5.3](#53-checksum-fields)).

## 3. Declarations and types

### 3.1. Top-level declarations and constants

The top-level declarations are `const`, `enum`, `bitfield`, `struct`, `union`, and `format`.

```rust
const MAX_PACKET_SIZE: u16 = 1500;
const PROTOCOL_VERSION: u8 = 1;
const MBR_SECTOR_SIZE: u16 = 512;

format DataPacket {
    version: u8 expect PROTOCOL_VERSION;
    payload_length: u16 where $value <= MAX_PACKET_SIZE;
    payload: bytes[payload_length];
}
```

Proposed rules for the initial design:

- A constant has an explicit type.
- Its initializer must be a compile-time constant expression.
- Constants are immutable and declared at the top level.
- Constants can be used in array lengths, expected values, constraints, conditional expressions, and compile-time assertions, provided the expression is valid in that context.
- A constant reference must resolve to a declared constant. Cyclic constant definitions are invalid.

The exact policy for forward references between constants can be finalized with the grammar and name-resolution design. The initial implementation may require constants to be declared before use.

### 3.2. Primitive types and arrays

The primitive types are fixed-width signed and unsigned integers and raw byte sequences:

- `u8`, `u16`, `u32`, `u64`
- `i8`, `i16`, `i32`, `i64`
- `bytes` - a raw byte sequence.

The width specifier `bits[N]` is available only for members of a `bitfield` declaration; it is not a standalone type that can be used directly in an ordinary `struct` or `format`.

Arrays use a uniform type-based syntax:

```rust
values: u16[8];
payload: bytes[length];
```

The array length can be a compile-time constant or an expression referencing fields that have already been parsed.

`bytes[$remaining]` represents the bytes left in the current bounded parsing region. Its detailed semantics are described in [Section 4.4](#44-remaining-bytes).

Floating-point types (`f32`, `f64`), strings, and more advanced collection types are not planned for the initial implementation.

### 3.3. Structures

A `struct` defines an ordered sequence of fields. Fields are parsed in declaration order. Their sizes and offsets are determined by their types, array lengths, conditional presence, and explicit padding.

```rust
struct PartitionEntry {
    boot_indicator: u8 where $value == 0x00 || $value == 0x80;
    starting_chs: bytes[3];
    partition_type: u8;
    ending_chs: bytes[3];
    starting_lba: u32;
    sector_count: u32;
}
```

A structure inherits the endianness of its enclosing context unless it specifies its own endianness. A field may also override the inherited value.

### 3.4. Enumerations

An enum has an explicitly specified integer underlying type and named integer values:

```rust
enum PartitionType: u8 {
    EMPTY  = 0x00;
    FAT12  = 0x01;
    FAT16  = 0x06;
    LINUX  = 0x83;
}
```

By default, enums are _closed_: a value not declared in the enum is rejected during runtime validation. An enum can be declared _open_ to preserve unknown underlying values:

```rust
enum MessageKind: u8 open {
    PING = 1;
    PONG = 2;
    DATA = 3;
}
```

An open enum does not make an unknown value a known named variant. Instead, the parser preserves its raw integer value so that a program can inspect it, report it, skip or handle an associated payload, or pass it onward without losing information.

The initial design should reject duplicate member names within an enum.

### 3.5. Unions and variant selection

A `union` defines named alternatives that describe different interpretations of the same data region:

```rust
struct TextPayload {
    encoding: u8;
    content: bytes[$remaining];
}

struct BinaryPayload {
    content: bytes[$remaining];
}

struct UnknownPayload {
    content: bytes[$remaining];
}

union Payload {
    text: TextPayload;
    binary: BinaryPayload;
    unknown: UnknownPayload;
}
```

The active alternative is selected using a `switch` field declaration. The discriminant must be available before the union field is parsed:

```rust
enum PayloadKind: u8 open {
    TEXT = 1;
    BINARY = 2;
}

format Message {
    kind: PayloadKind;

    payload: Payload switch kind {
        TEXT => text;
        BINARY => binary;
        default => unknown;
    };
}
```

Proposed `switch` rules:

- The `switch` expression references a field that has already been parsed.
- Each case maps a constant discriminant value to an alternative declared by the specified union.
- A case value must be compatible with the discriminant's type; duplicate case values are invalid.
- `default` is optional and selects an alternative when no explicit case matches.
- If no case matches and there is no `default`, parsing fails with an unsupported-variant diagnostic.
- Exactly one alternative is selected for a given input value.

The example uses `default => unknown` to preserve the payload of a message with an unrecognized kind. The `bytes[$remaining]` field inside the selected alternative consumes the rest of the bounded payload region.

### 3.6. Bitfield types

A `bitfield` declaration defines one fixed-width unsigned storage unit divided into named fields. The storage type must be one of `u8`, `u16`, `u32`, or `u64`:

```rust
bitfield TcpControl: u16 {
    bit_order msb_first;

    data_offset: bits[4];
    reserved: bits[4];
    cwr: bits[1];
    ece: bits[1];
    urg: bits[1];
    ack: bits[1];
    psh: bits[1];
    rst: bits[1];
    syn: bits[1];
    fin: bits[1];
}
```

The declaration describes a 16-bit value. The sum of the member widths must equal the width of the underlying storage type. A member is an unsigned integer value whose width is specified by `bits[N]`; `expect` and `where` constraints may be applied to an individual member. These constraints are evaluated against the extracted member value during runtime validation; inside a member's `where` constraint, `$value` refers to that member value.

Every `bitfield` declaration must contain exactly one `bit_order` directive:

- `msb_first` assigns the first declared member to the most-significant available bits of the storage value, then continues toward the least-significant bits.
- `lsb_first` assigns the first declared member to the least-significant available bits, then continues toward the most-significant bits.

Bit positions are defined on the numeric storage value after its bytes have been interpreted using endianness. For example, in a `u16` bitfield with `msb_first`, the first four-bit member occupies numeric bits 15 through 12. 

A bitfield is used as an ordinary type in a structure or format:

```rust
format TcpHeader {
    endian big;

    source_port: u16;
    destination_port: u16;
    sequence_number: u32;
    acknowledgment_number: u32;
    control: TcpControl;
    window: u16;
    checksum: u16;
    urgent_pointer: u16;
}
```
The same mechanism can describe the packed sector/cylinder portion of an MBR CHS address:

```rust
bitfield CHSSectorCylinder: u8 {
    bit_order msb_first;

    cylinder_high: bits[2];
    sector: bits[6];
}

struct CHSAddress {
    head: u8;
    sector_cylinder: CHSSectorCylinder;
    cylinder_low: u8;
}
```

Individual members are accessed through the containing field, for example `control.ack`. A field-level `endian` modifier applies to the complete storage unit, not to an individual bitfield member. If it is omitted, as for `control` in `TcpHeader`, the bitfield instance inherits the enclosing endianness context.

Initial bitfield rules:

- The storage type is exactly one of `u8`, `u16`, `u32`, or `u64`.
- Every member uses `bits[N]`, where `N` is positive and does not exceed the storage width.
- The sum of all member widths must equal the storage width exactly; there are no implicit unused bits.
- Members are packed contiguously in declaration order according to `bit_order`.
- A member may cross a byte boundary within its storage unit, but it cannot extend beyond the storage unit. Members are never packed across separate bitfield declarations.
- Signed bitfield members and nested bitfield declarations are not supported initially.
- `offset_of` returns the byte offset of the complete bitfield instance. A built-in for the bit offset of an individual member is not part of the initial scope.

The C generator must not rely on implementation-defined C bit-field layout. It will read the underlying unsigned integer using explicit byte-order handling and extract members with masks and shifts. Generated public data structures can expose the extracted members as ordinary fixed-width unsigned integer values.

## 4. Field and layout constructs

### 4.1. Endianness

Endianness can be specified at the `struct` or `format` level:

```rust
format NetworkPacket {
    endian big;
    // ...
}
```

Fields inherit the endianness of their enclosing context. A field or nested structure can override it:

```rust
value: u32 endian little;
```

The supported values are `little` and `big`. Endianness does not apply to raw byte sequences.

### 4.2. Field-level constraints

Fields can have an optional endianness modifier and one field-level value constraint.

`expect` - expected value

```rust
signature: bytes[2] expect hex("55 AA");
version: u8 expect 1;
```

The parsed value must equal the specified value. The expected expression must be a compile-time constant compatible with the field type.

`where` - value condition

```rust
length: u16 where $value > 0;
flags: u8 where ($value & 0xF0) == 0;
```

The built-in `$value` refers to the value of the current field. Names that start with `$` are provided by the language, and user-declared names cannot start with `$`, so `$value` never conflicts with a field that happens to be named `value`. The expression must evaluate to a boolean value at runtime.

For the initial design, `expect` and `where` are mutually exclusive on an individual field. Cross-field conditions belong in a `validate` block instead.

### 4.3. Conditional fields

The `if` construct allows fields to be present only when a condition is satisfied:

```rust
struct Packet {
    flags: u8;

    if (flags & 0x01) != 0 {
        sequence_number: u32;
    }

    payload: bytes[$remaining];
}
```

Conditions can reference previously declared fields and constants. Conditional blocks can contain ordinary field declarations and follow the same field rules. Nested conditional blocks are excluded from the initial scope.

A field inside a conditional block has no runtime value when its condition is false. The built-in `present(field)` tests whether a conditional field was parsed: it is true when the field is present and false when it is absent. For a field that is not inside an `if` block, it is always true.

### 4.4. Remaining bytes

`$remaining` is a built-in value: the number of bytes between the start of the current field and the end of the current bounded parsing region. It is evaluated at parse time, at the point where the field begins. `bytes[$remaining]` consumes all bytes remaining in the current bounded parsing region.

Examples of bounded regions include:

- the input region supplied to a top-level `format` parser;
- a nested field whose length is determined by its parent;
- a selected union alternative whose containing field has a known boundary.

A `bytes[$remaining]` field must be the final data-consuming field of its bounded region. No subsequent field in that same region may require bytes after it. If the parser cannot determine the boundary of the current region, use of `$remaining` must produce a diagnostic rather than silently consume an undefined amount of input.

In the initial design, `$remaining` is only allowed as the complete length of a `bytes` field, as in `bytes[$remaining]`. Using it inside a larger expression, for example `bytes[$remaining - 4]` followed by a fixed-size trailer such as a CRC, is a planned extension.

This rule allows a nested payload to consume its own region without accidentally consuming bytes belonging to an enclosing or subsequent structure.

### 4.5. Padding and layout

Explicit padding is represented by a dedicated declaration:

```rust
struct Header {
    magic: bytes[4];
    version: u8;
    pad[3];
    timestamp: u64;
}
```

Padding occupies the specified number of bytes without introducing an ordinary named data field. The initial semantics should treat padding as bytes to skip; whether padding contents must be zero or satisfy another pattern can be considered separately.

Static layout analysis calculates field sizes and offsets whenever they can be determined. More advanced layout features, such as explicit offsets and alignment directives, are planned extensions.

### 4.6. Repeated elements
 
When the number of elements is not known in advance, an array field can repeat while or until a condition holds instead of using a count. Empty brackets (`Type[]`) are only valid together with a `while` or `until` clause, which decides how many elements are parsed.
 
```rust
struct Chunk {
    length: u32;
    chunk_type: bytes[4];
    data: bytes[length];
}
 
enum OptionKind: u8 open {
    END = 0;
    NOP = 1;
}
 
struct Option {
    kind: OptionKind;
    length: u8;
    data: bytes[length];
}
 
format ChunkList {
    endian big;
 
    chunks: Chunk[] while $remaining > 0;
}
 
format OptionList {
    options: Option[] until $last.kind == OptionKind.END;
}
 
format CString {
    name: u8[] until $last == 0;
}
```
 
`ChunkList` repeats `Chunk` until the input is exhausted, `OptionList` repeats `Option` until an element with the `END` kind has been parsed (that element is included), and `CString` collects bytes up to and including a zero byte.
 
- `while condition` evaluates the condition before each element and parses another element only while it is true. The array can therefore have no elements. `while $remaining > 0` repeats until the bounded region is exhausted.
- `until condition` evaluates the condition after each element and stops as soon as it is true. The array has at least one element, and the terminating element is part of the array.
- Inside an `until` condition, `$last` is the element that was just parsed. 
- `$remaining` can be used in the condition of either form. It is the number of bytes between the current position and the end of the bounded region.

Rules: 
- Each element is decoded and checked as described in [Section 7.1](#71-evaluation-order-and-errors) before the condition is evaluated.
- Each element must consume at least one byte, otherwise the loop could never advance. Layout analysis rejects element types whose size can be zero.
- Every read is bounds-checked and every element consumes at least one byte, so the number of iterations is limited by the size of the bounded region. If the region ends in the middle of an element, or before an `until` condition becomes true, parsing fails with a truncated-input error.
- A repeated array has a runtime-dependent length, so the offsets of the fields after it are not statically known.


### 4.7. Size-bounded fields
 
A `size` clause gives a field an explicit length in bytes. The parser sets aside exactly that many bytes as a bounded region, parses the field inside the region, and continues with the next field right after it.

```rust
format Message {
    endian big;
 
    kind: PayloadKind;
    length: u16;
    payload: Payload size length switch kind {
        TEXT => text;
        BINARY => binary;
        default => unknown;
    };
    crc: u32;
}
```

The payload occupies exactly `length` bytes. 
 
Rules:
- `size expression` immediately follows the field type. The expression is an integer expression that can reference constants and fields that have already been parsed.
- `size` can be used on fields of struct, union, and array types. It is not allowed on integers, enums, bitfields, bytes, or checksum fields, whose length is already determined by their type.
- The region is the bounded region of the field: `$remaining` counts to the end of the region (see [Section 4.4](#44-remaining-bytes)).
- The field must consume exactly the number of bytes given by `size`. If it consumes fewer, parsing fails with a size-mismatch error; if it needs more, parsing fails with a truncated-input error. To accept and keep extra bytes, end the field with `bytes[$remaining]`.
- The size expression is evaluated after the fields it references have been decoded and checked (see [Section 7.1](#71-evaluation-order-and-errors)). If it evaluates to a negative number, or to more bytes than remain in the enclosing region, parsing fails before the field is parsed.
- With an expression that depends on parsed fields, the field is variable-length, and the offsets of the fields after it are not statically known.
- `size` can be combined with `switch`, `endian`, `while`, and `until` as shown in [Section 8](#8-field-modifier-compatibility).

## 5. Validation and checksums

### 5.1. Validation blocks

A `validate` block defines constraints involving multiple fields of the enclosing `struct` or `format`:

```rust
struct PacketHeader {
    header_size: u16 endian big;
    payload_size: u16 endian big;
    total_size: u32 endian big;

    validate {
        total_size == header_size + payload_size;
        total_size <= 65535;
    }
}
```

Each expression must evaluate to a boolean value. All expressions must be satisfied for the parsed structure to be considered valid.

Rules:

- At most one `validate` block per `struct` or `format`.
- The block appears after field declarations and other declaration-level checks.
- Expressions can reference fields belonging to the enclosing declaration, subject to conditional-presence rules.
- The block contains boolean expressions only, not field declarations or general-purpose statements.

Unlike `where`, which is intended for an individual field, `validate` is intended for cross-field constraints and structural invariants.

### 5.2. Compile-time assertions

The `assert` construct verifies properties of the specification itself, such as sizes and offsets:

```rust
struct CommonHeader {
    version: u8;
    kind: u8;
    reserved: u8;

    assert size_of(Self) == 3;
}

format DataPacket {
    magic: bytes[2];
    header: CommonHeader;
    flags: u8;
    payload_length: u16;

    assert offset_of(payload_length) == 6;
}
```

Assertions are evaluated during specification analysis, not against parsed binary input.

Proposed built-in functions:

- `size_of(T)` - obtains the statically known size of a type.
- `offset_of(field)` - obtains the statically known offset of a field within the enclosing declaration.

An assertion whose requested size or offset cannot be determined statically must produce a diagnostic.

### 5.3. Checksum fields

Checksum fields use a dedicated field modifier followed by a configuration block:

```rust
crc: u32 using crc32 {
    covers = [chunk_type, data];
}
```

The checksum is calculated over the selected fields and compared with the value stored in the binary input.

Rules:

- The checksum algorithm must be one of the algorithms built into BFSL.
- The `covers` list references fields declared before the checksum field.
- The checksum field cannot include itself in its own calculation.
- The bytes passed to the algorithm are the original input bytes of each covered field, exactly as they are encoded in the input.
- The field order in `covers` determines the order in which the byte sequences are supplied to the algorithm.
- A conditionally absent covered field contributes no bytes.
- The checksum field can specify an endianness modifier; otherwise it inherits the current endianness context.
- A checksum field cannot additionally use `where` or `expect`.

The initial implementation should support one precisely defined CRC-32 variant (CRC-32/ISO-HDLC). Additional checksum algorithms can be introduced later.

## 6. Expressions and built-ins

Expressions are intentionally restricted. The operators are:

- Arithmetic: `+`, `-`, `*`, `/`, `%`
- Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
- Logical: `&&`, `||`, `!`
- Bitwise: `&`, `|`, `^`, `~`, `<<`, `>>`
- Grouping: parentheses

Names that start with `$` are reserved for built-in values supplied by the language; user-declared names cannot start with `$`.

The initial built-ins are:

| Built-in | Purpose | Evaluation phase |
|---|---|---|
| `hex("...")` | Converts a hexadecimal string into a byte sequence for literals and expected values. | Compile time |
| `ascii("...")` | Converts a string of printable ASCII characters into a byte sequence with one byte per character, for literals such as magic numbers and chunk types, expected values, and `switch` case labels. | Compile time |
| `zeros(n)` | Returns a byte sequence of `n` zero bytes, where `n` is a non-negative constant. Used for expected values of reserved bytes. | Compile time |
| `size_of(T)` | Obtains the statically known size of a type. | Compile time, after layout analysis |
| `offset_of(field)` | Obtains the statically known offset of a field. | Compile time, after layout analysis |
| `len(array)` | Returns the number of elements of an array field that has already been parsed; for a `bytes` field, the number of bytes. | Runtime |
| `present(field)` | Returns whether a conditionally present field was parsed. See [Section 4.3](#43-conditional-fields). | Runtime |
| `$value` | The decoded value of the field whose `where` constraint is being evaluated. Valid only inside `where`. | Runtime |
| `$remaining` | The number of bytes from the start of the current field to the end of the current bounded region. In the initial design, valid only as the complete length of a `bytes` field. | Runtime |
| `$last` | The element that was just parsed. Valid only inside the `until` condition of a repeated array. | Runtime |



## 7. Semantic analysis and evaluation

BFSL distinguishes between specification-time checks and runtime validation of binary input.

The proposed processing stages are:

1. Parsing and model construction: read the specification through the textual or UML frontend and convert it into the common IR (see [Section 9](#9-intermediate-representation)).
2. Declaration and symbol analysis: register constants and type declarations, resolve names and field references, detect duplicate or unresolved declarations, and validate declaration relationships.
3. Type and expression analysis: verify expression types, operand compatibility, constant-expression requirements, and field availability in each context.
4. Layout analysis: calculate field sizes and offsets whenever possible, determine bounded regions, and evaluate compile-time assertions.
5. Runtime-check construction: represent `expect`, `where`, `validate`, enum-value checks, switch selection, and checksum verification in the analyzed model.
6. Code generation: translate the analyzed model into a target implementation, initially C parser and validator code.

Stages 2 to 4 together form *semantic analysis*; layout analysis (stage 4) is its final pass.

The main constructs have different roles and evaluation stages. In the table below, compile time means specification analysis performed by the BFSL toolchain (not the compilation of generated C code), parsing means decisions about how the input bytes are read (structure, presence, selection), and runtime means value checks performed by the generated parser on binary input:

| Construct | Purpose | Evaluation |
|---|---|---|
| `const` | Defines a reusable immutable value. | Compile time |
| `endian` | Determines byte order when interpreting integer values. | Parsing |
| `expect` | Requires a specific field value. | Runtime |
| `where` | Requires a field value to satisfy a condition. | Runtime |
| `if` | Determines whether conditional fields are present. | Parsing |
| `switch` | Selects a union alternative based on a discriminant. | Parsing |
| `while` / `until` | Repeats an array element while, or until, a condition holds. | Parsing |
| `size` | Gives a field an explicit length in bytes and makes that range its bounded region. | Parsing |
| `using` | Defines checksum calculation and verification. | Runtime |
| `bit_order` | Determines how bitfield members are assigned positions within the numeric storage value. | Compile time; extraction during parsing |
| `validate` | Checks relationships between fields. | Runtime |
| `assert` | Checks properties of the specification and layout. | Compile time |

For field-dependent expressions, array lengths and conditional expressions initially reference only previously declared fields. `where` uses `$value`, the value of the current field; `validate` can reference fields of the enclosing declaration, subject to the rules for conditionally present fields.

### 7.1. Evaluation order and errors

Fields are processed in declaration order. Each field is decoded, checked, and only then made available to the fields that follow it:

1. The field is read from the current bounded region. A read that would go past the end of the region fails with a truncated-input error; no field is ever read out of bounds.
2. Its own constraints are evaluated immediately: `expect`, `where`, the closed-enum check, and the constraints of bitfield members.
3. Only after these checks succeed can its value be used by later fields, for example as an array length, in an `if` condition, or as a `switch` discriminant.

A value that fails its constraints is therefore never used to size, select, or skip later data. Checks that must hold before a value is used by later fields (such as an upper bound on a length field) should be written with `where`; `validate` is intended for relationships between several fields.

The remaining checks run when all the data they depend on has been parsed:

- A `validate` block is evaluated after all fields of its enclosing `struct` or `format` have been parsed. For a nested structure, this happens before the parent continues with its next field.
- A checksum is verified once the checksum field and all the fields it covers have been parsed.

Parsing stops at the first failure (fail-fast). The error identifies the kind of failure, the field path, and the byte offset in the input. Collecting several errors in a single pass is not part of the initial scope.

### 7.2. Input region and trailing bytes

A `format` is parsed from the beginning of the input region supplied by the caller and stops after its last field. Bytes that follow the end of the format are neither an error nor a warning. The generated API reports how many bytes were consumed, and the caller decides what to do with the rest. This allows a format to be parsed from a larger buffer, for example a header followed by its payload, or one chunk of a sequence of chunks.

A format whose last field is `bytes[$remaining]` consumes the whole region, so no trailing bytes remain. If the input ends before the format is complete, parsing fails with a truncated-input error.

## 8. Field modifier compatibility

The ordinary field syntax is intentionally restrictive. In the forms below, square brackets around a modifier mark it as optional; they are not part of the syntax.

```text
field_name: Type [size expression] [endian] [expect | where];
```

The `size` clause applies only to fields of struct, union, and array types; see [Section 4.7](#47-size-bounded-fields). It is not available on checksum fields.

A `switch` field uses a dedicated form:

```text
field_name: UnionType switch discriminant {
    constant => alternative;
    default => alternative;
};
```

Checksum fields use another dedicated form:

```text
field_name: Type [endian] using algorithm {
    covers = [field_a, field_b];
}
```

Repeated arrays use a `while` or `until` clause instead of a count:
 
```text
field_name: Type[] [size expression] [endian] while condition;
field_name: Type[] [size expression] [endian] until condition;
```
 
Repeated arrays are not combined with `expect`, `where`, `switch`, or `using`; constraints on the elements belong to the element type.

`switch` is not combined with `expect` or `where` in the initial design. Its selection and unknown-case behavior are defined by the union and switch rules.

Bitfield members use a distinct declaration form, `member_name: bits[N] [expect | where];`. They do not have individual endianness modifiers; the containing bitfield instance interprets the complete storage unit using its inherited or explicitly overridden endianness.

## 9. Intermediate Representation

BFSL will use a common, frontend-independent Intermediate Representation (IR) as the central model of the toolchain. Both the textual frontend, implemented using textX, and the UML frontend, based on a custom UML Profile and an XMI adapter, will transform their input models into this representation.

The IR will contain normalized type declarations, constants, fields, resolved references, expressions, constraints, switch mappings, checksum configuration, and format-specific information. It will not depend on the internal representation of either frontend.

The processing pipeline is built around semantic analysis, which ends with a layout-analysis pass. The earlier passes will resolve types and references, verify expression types, check constant values and switch mappings, and detect invalid declarations. The layout pass will calculate field sizes, offsets, and bounded regions whenever possible, and evaluate compile-time assertions.

The analyzed IR will serve as the input for code generators. This allows multiple frontends and future code generators to share the same semantic model and analysis infrastructure. The initial backend is C; additional targets such as Python are possible future extensions, not initial deliverables.

The following simplified example illustrates the intended separation between source syntax and the normalized IR. It is not a finalized IR schema or serialization format.

BFSL source:

```rust
const MAX_PAYLOAD_SIZE: u16 = 4096;

format DataPacket {
    endian big;
    magic: bytes[2] expect hex("BF 01");
    payload_length: u16;
    payload: bytes[payload_length];

    validate {
        payload_length <= MAX_PAYLOAD_SIZE;
    }
}
```

Illustrative normalized IR:

```yaml
module:
  constants:
    - id: MAX_PAYLOAD_SIZE
      type: u16
      value: 4096
  formats:
    - id: DataPacket
      endian: big
      fields:
        - id: magic
          type: bytes
          length: 2
          constraints:
            - kind: equals
              value: [0xBF, 0x01]
        - id: payload_length
          type: u16
        - id: payload
          type: bytes
          length:
            kind: field_reference
            target: payload_length
      validations:
        - kind: less_equal
          left: payload_length
          right:
            kind: constant_reference
            target: MAX_PAYLOAD_SIZE
```

The frontend adapters resolve the source-level declarations and references into normalized IR nodes. The first semantic-analysis passes verify their types and relationships; the layout pass then annotates the IR with statically known sizes and offsets. For this example, `magic` has offset 0 and size 2, `payload_length` has offset 2 and size 2, while the payload starts at offset 4 and has a runtime-dependent length. The IR schema itself will be refined during implementation.

A bitfield declaration is represented as a type with a storage type, bit order, and members whose widths and bit offsets are explicit after layout analysis. For example, the `TcpControl` declaration above can be lowered to the following illustrative fragment (bit offset 0 denotes the least-significant bit of the numeric storage value):

```yaml
- kind: bitfield
  id: TcpControl
  storage_type: u16
  bit_order: msb_first
  size_bytes: 2
  members:
    - { id: data_offset, width: 4, bit_offset: 12 }
    - { id: reserved, width: 4, bit_offset: 8 }
    - { id: cwr, width: 1, bit_offset: 7 }
    - { id: ece, width: 1, bit_offset: 6 }
    - { id: urg, width: 1, bit_offset: 5 }
    - { id: ack, width: 1, bit_offset: 4 }
    - { id: psh, width: 1, bit_offset: 3 }
    - { id: rst, width: 1, bit_offset: 2 }
    - { id: syn, width: 1, bit_offset: 1 }
    - { id: fin, width: 1, bit_offset: 0 }
```

## 10. UML frontend and UML Profile

The graphical frontend will be implemented using a custom UML Profile in MagicDraw, extending existing UML metaclasses with BFSL-specific stereotypes and tagged values. The profile will allow users to describe binary formats through familiar UML elements while preserving the semantics of the textual language.

The initial profile is expected to include the following stereotypes:

| Stereotype | Extended UML metaclass | Purpose |
|---|---|---|
| `BFSLFormat` | Class | Represents a top-level binary format. |
| `BFSLStruct` | Class | Represents a composite structure. |
| `BFSLUnion` | Class | Represents alternative interpretations of binary data. |
| `BFSLBitfield` | Class | Represents a fixed-width storage unit divided into bit members. |
| `BFSLEnum` | Enumeration | Represents an integer-based enumeration. |
| `BFSLField` | Property | Represents a field in a binary structure. |
| `BFSLPadding` | Property | Represents explicit padding. |
| `BFSLChecksum` | Property | Represents a checksum field. |
| `BFSLValidation` | Constraint | Represents runtime validation constraints. |
| `BFSLAssertion` | Constraint | Represents compile-time layout assertions. |

Tagged values will be used to specify properties such as endianness, array lengths, expected values, field conditions, switch mappings, enum openness, and checksum configuration. Constants may be represented as suitable UML properties or stereotype applications.

An XMI adapter will transform UML models into the common BFSL IR. Consequently, the textual and graphical frontends will share semantic analysis, layout analysis, and code generation components. The UML frontend must not introduce semantics that cannot be represented by the textual frontend or common IR.

The proposed UML Profile and its mapping to the IR will be refined during implementation.

## Appendix A. Reference tables
 
This appendix collects the keywords, types, built-ins, operators and checksum algorithms of BFSL in one place. The sections of the main text define their meaning.

### A.1. Keywords and names

BFSL has three kinds of special names:
 
- Keywords are reserved: they cannot be used as the name of a declaration, field, constant, or enum member. The list is fixed by the language version.
- Built-in functions, such as `hex` and `size_of`, are recognized only in a call position, that is, a name followed by `(`. They are not reserved, so new built-in functions can be added without breaking existing specifications.
- Built-in values start with `$`, such as `$value`. User-declared names cannot start with `$`.

| Keyword | Category | Meaning | Section |
|---|---|---|---|
| `const` | Declaration | Defines a compile-time constant. | [3.1](#31-top-level-declarations-and-constants) |
| `enum` | Declaration | Defines an enumeration. | [3.4](#34-enumerations) |
| `bitfield` | Declaration | Defines a fixed-width storage unit divided into bit members. | [3.6](#36-bitfield-types) |
| `struct` | Declaration | Defines a helper structure used by formats. | [3.3](#33-structures) |
| `union` | Declaration | Defines alternative layouts, of which one is selected. | [3.5](#35-unions-and-variant-selection) |
| `format` | Declaration | Defines a top-level binary format, which is a parsing entry point. | [3.3](#33-structures) |
| `open` | Declaration modifier | Marks an enum as open. | [3.4](#34-enumerations) |
| `endian` | Statement and field clause | Sets or overrides the byte order. | [4.1](#41-endianness) |
| `bit_order` | Statement | Sets the bit order of a bitfield. | [3.6](#36-bitfield-types) |
| `expect` | Field clause | Requires an exact value. | [4.2](#42-field-level-constraints) |
| `where` | Field clause | Requires a condition on the value. | [4.2](#42-field-level-constraints) |
| `size` | Field clause | Gives a field an explicit length in bytes. | [4.7](#47-size-bounded-fields) |
| `switch` | Field clause | Selects a union alternative. | [3.5](#35-unions-and-variant-selection) |
| `while` | Field clause | Repeats an array element while a condition holds. | [4.6](#46-repeated-elements) |
| `until` | Field clause | Repeats an array element until a condition holds. | [4.6](#46-repeated-elements) |
| `using` | Field clause | Introduces the checksum configuration. | [5.3](#53-checksum-fields) |
| `covers` | Checksum block | Lists the fields included in a checksum. | [5.3](#53-checksum-fields) |
| `if` | Statement | Makes a group of fields conditional. | [4.3](#43-conditional-fields) |
| `default` | `switch` case | Selects the fallback alternative. | [3.5](#35-unions-and-variant-selection) |
| `pad` | Statement | Declares explicit padding. | [4.5](#45-padding-and-layout) |
| `assert` | Statement | Declares a compile-time assertion. | [5.2](#52-compile-time-assertions) |
| `validate` | Block | Declares cross-field validation. | [5.1](#51-validation-blocks) |
| `bits` | Type | Width of a bitfield member. | [3.6](#36-bitfield-types) |
| `bytes` | Type | Raw byte sequence. | [3.2](#32-primitive-types-and-arrays) |
| `Self` | Type name | The enclosing declaration, as in `size_of(Self)`. | [5.2](#52-compile-time-assertions) |
| `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64` | Type | Primitive integer types. | [3.2](#32-primitive-types-and-arrays) |
 
Some words have a special meaning only in one position and are not reserved: `little` and `big` after `endian`, `msb_first` and `lsb_first` after `bit_order`, and `crc32` after `using`.

### A.2. Types
 
| Type | Description | Size |
|---|---|---|
| `u8`, `u16`, `u32`, `u64` | Unsigned integers. | 1, 2, 4, and 8 bytes |
| `i8`, `i16`, `i32`, `i64` | Signed integers in two's complement. | 1, 2, 4, and 8 bytes |
| `bytes[N]` | Raw byte sequence of `N` bytes. `bytes[$remaining]` takes the rest of the bounded region. | `N` bytes |
| `Type[count]` | Array with a given number of elements of an integer, enum, bitfield, or struct type. | `count` times the element size |
| `Type[]` | Array repeated with a `while` or `until` clause. | Depends on the input |
| `bits[N]` | Member of a bitfield declaration; not usable in structs or formats. | `N` bits |
| Declared types | Enums, bitfields, structs, and unions. A union field needs a `switch`, and a format cannot be used as a field type. | See the declaration |
 
Integers of more than one byte are interpreted using the endianness in effect ([Section 4.1](#41-endianness)). Raw byte sequences have no endianness.

### A.3. Built-in functions and values
 
| Name | Kind | Arguments and result | Where it can be used | Phase | Section |
|---|---|---|---|---|---|
| `hex("...")` | Function | A string of hexadecimal digit pairs (spaces allowed, case ignored); returns `bytes`. | Constant byte sequences: `expect`, comparisons, and `switch` labels. | Compile time | [6](#6-expressions-and-built-ins) |
| `ascii("...")` | Function | A string of printable ASCII characters; returns `bytes` with one byte per character. | Same as `hex`. | Compile time | [6](#6-expressions-and-built-ins) |
| `size_of(T)` | Function | A type or `Self`; returns its size in bytes. | `assert`. | Compile time, after layout analysis | [5.2](#52-compile-time-assertions) |
| `offset_of(field)` | Function | A field of the enclosing declaration; returns its offset in bytes. | `assert`. | Compile time, after layout analysis | [5.2](#52-compile-time-assertions) |
| `len(array)` | Function | An array field or `bytes` field that has already been parsed; returns its number of elements (bytes, for `bytes`). | Any expression that can reference an already parsed field. | Runtime | [4.6](#46-repeated-elements) |
| `present(field)` | Function | A field of the enclosing declaration; returns a boolean that is true when the field is present. | Any expression, subject to the reference rule in Section 4.3. | Runtime | [4.3](#43-conditional-fields) |
| `$value` | Value | The decoded value of the field or bitfield member that the constraint belongs to. | Only inside `where`. | Runtime | [4.2](#42-field-level-constraints) |
| `$remaining` | Value | The number of bytes between the current position and the end of the bounded region. | As the complete length of a `bytes` field, and in `while` and `until` conditions. | Runtime | [4.4](#44-remaining-bytes) |
| `$last` | Value | The array element that was just parsed. | Only inside `until`. | Runtime | [4.6](#46-repeated-elements) |

### A.4. Operators
 
Operators are listed from the highest to the lowest precedence. The precedence and associativity below are proposed for the initial design.
 
| Level | Operators | Associativity | Operands |
|---|---|---|---|
| 1 | `.` member access, `f(...)` call | Left | Field, bitfield member, or enum member (`OptionKind.END`) |
| 2 | `!`, `~` (unary) | Right | `!` boolean, `~` integer |
| 3 | `*`, `/`, `%` | Left | Integers |
| 4 | `+`, `-` | Left | Integers |
| 5 | `<<`, `>>` | Left | Integers |
| 6 | `&` | Left | Integers |
| 7 | `^` | Left | Integers |
| 8 | `\|` | Left | Integers |
| 9 | `==`, `!=`, `<`, `>`, `<=`, `>=` | None; cannot be chained | Integers. `==` and `!=` also compare byte sequences of the same length and enum values of the same enum. |
| 10 | `&&` | Left | Booleans |
| 11 | `\|\|` | Left | Booleans |

Conditions in `if`, `where`, `validate`, `assert`, `while`, and `until` must be boolean. Integers are not converted to booleans implicitly; write `flags.sequence_present != 0`.

### A.5. Checksum algorithms
 
| Name | Definition | Width | Section |
|---|---|---|---|
| `crc32` | CRC-32/ISO-HDLC: polynomial `0x04C11DB7`, reflected input and output, initial value `0xFFFFFFFF`, final XOR `0xFFFFFFFF`. The check value for the ASCII bytes `123456789` is `0xCBF43926`. | 32 bits | [5.3](#53-checksum-fields) |
 
Additional algorithms can be introduced later; each one is identified by its name after `using`.

### A.6. Literals, comments, and names
 
- Integer literals are written in decimal (`4096`) or in hexadecimal with a `0x` prefix (`0xBF01`).
- String literals are written in double quotes and are used only as arguments of `hex` and `ascii`.
- A comment starts with `//` and runs to the end of the line.
- Names consist of letters, digits, and underscores, do not start with a digit or `$`, and are case-sensitive. The examples in this document use `UpperCamelCase` for types, `snake_case` for fields, and `UPPER_SNAKE_CASE` for constants and enum members; this is a convention and is not enforced by the grammar.