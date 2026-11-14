# BFSL - Project Proposal

## Introduction and motivation

Binary formats appear in many areas of software, including file formats, disk structures, and network protocols. They are typically defined in specification documents as a sequence of fields with fixed or data-dependent sizes, a byte order, and rules that valid data must satisfy. A parser for such a format is then usually written by hand in a general-purpose language.

Hand-written parsers of this kind consist largely of repetitive low-level code: reading fields of a given width, converting byte order, checking lengths against the input boundary, and verifying expected values and checksums. Such code is prone to errors such as missing bounds checks or incorrect byte order, and the structure of the format is no longer directly visible once it has been translated into parsing logic. Any change to the format requires locating and updating every part of the code that depends on it.

We propose a domain-specific language in which the structure and validation rules of a binary format are described declaratively, and from which the parsing and validation code is generated automatically.

## What BFSL does

BFSL (Binary Format Specification Language) is the declarative language for describing binary formats. In a `.bfsl` file the developer would describe:

- which fields the format has, in what order, and of what type (`u8`, `u16`, `u32`, raw bytes, arrays, nested structures);
- the byte order (`endian big` / `endian little`);
- how lengths and the presence of fields depend on values read earlier (a length field that sets the size of the next field, a flag that turns a field on or off, a type tag that selects which structure follows);
- what makes the data valid: expected values (magic numbers), allowed ranges, relationships between fields, and checksums.

From this description, the BFSL tool we plan to build should:

1. check the specification itself and report mistakes (unknown names, a field used before it is read, bitfields whose widths do not add up, a layout that does not match an asserted size);
2. generate a **C parser and validator** that reads a byte buffer, fills a C struct with the decoded fields, and rejects invalid or truncated input with an error that names the field and byte offset;
3. provide **editor support** in VS Code through textX-LS: syntax highlighting, completion, go-to-definition, and the error messages from step 1 shown directly in the editor.

We plan to implement the language using **textX** and to package the project as an installable Python package with a `bfsl` command (for example `bfsl check` and `bfsl generate`). The language and the generator will also be registered with textX, so that they can be used through the `textx` command as well.

### Connection with our other course

The same project is also our topic for the Rapid Software Development Methodologies course. There we plan to build a **graphical** way to describe the same formats (a UML Profile in MagicDraw), while in this course we propose the **textual** language. To keep the two from turning into two separate tools, both will be translated into one common intermediate representation (IR). Checking and C code generation will work only on that IR, so they can be shared:

```text
 .bfsl file  --(textX)-------->  \
                                  >  common IR  -->  checks  -->  C code generator
 UML model   --(XMI adapter)-->  /
```

For this course the relevant parts are the textual language, its grammar, the editor support, the checks, and the C generator. **The UML frontend belongs to the other course and is mentioned here only because it is the reason the IR is planned as a separate layer.**

## Representative example

Below is how we imagine a description of a PNG file would look in the proposed syntax. A PNG file starts with a fixed 8-byte signature, followed by a list of chunks until the end of the file. Every chunk has a length, a four-letter type, `length` bytes of data, and a CRC-32 computed over the type and the data. The first chunk, `IHDR`, has a fixed layout that we also want to check.

```rust
enum ColorType: u8 {
    GRAYSCALE       = 0;
    RGB             = 2;
    PALETTE         = 3;
    GRAYSCALE_ALPHA = 4;
    RGBA            = 6;
}

// Contents of the IHDR chunk: image width, height and pixel format.
struct ImageHeader {
    width: u32 where $value > 0;
    height: u32 where $value > 0;
    bit_depth: u8;
    color_type: ColorType;      // closed enum: any other value is rejected
    compression: u8 expect 0;
    filter: u8 expect 0;
    interlace: u8 where $value <= 1;

    assert size_of(Self) == 13; // checked when the specification is compiled
}

// Any other chunk: we keep its bytes without interpreting them.
struct RawChunk {
    content: bytes[$remaining];
}

union ChunkData {
    header: ImageHeader;
    other: RawChunk;
}

struct Chunk {
    length: u32 where $value <= 0x7FFFFFFF;
    chunk_type: bytes[4];

    // Exactly `length` bytes, interpreted depending on the chunk type.
    data: ChunkData size length switch chunk_type {
        ascii("IHDR") => header;
        default => other;
    };

    crc: u32 using crc32 {
        covers = [chunk_type, data];
    }
}

format PngFile {
    endian big;

    signature: bytes[8] expect hex("89 50 4E 47 0D 0A 1A 0A");
    chunks: Chunk[] while $remaining > 0;
}
```

Two things are worth pointing out. Because `data` must consume exactly `length` bytes, an `IHDR` chunk whose length is not 13 would be rejected without any extra rule. And the checks happen at two different times: `assert` is checked against the specification itself, before any C code is generated, while `expect`, `where`, the enum values and the CRC are checked by the generated parser against the actual input.

From this specification, the code generator would produce a C header and source file. The following snippet illustrates the intended use of the generated parser:

```c
PngFile png;
bfsl_error err;

if (PngFile_parse(buffer, buffer_len, &png, &err) != BFSL_OK) {
    printf("invalid PNG: %s at field %s, offset %zu\n",
           err.message, err.field_path, err.offset);
    return 1;
}

printf("image %u x %u, %zu chunks\n",
       png.chunks[0].data.header.width,
       png.chunks[0].data.header.height,
       png.chunks_count);
```

The developer would not have to write the bounds checks, byte swapping, CRC calculation, or length bookkeeping by hand; all of it would follow from the specification.

The full language definition, with all constructs and rules, is in [`docs/language-design.md`](docs/language-design.md). That document is provided for reference only; reading it is not necessary to understand this proposal.
