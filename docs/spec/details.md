# Implementation Details

## NOP tags and Alignment
Alignment refers to the position of an element with respect to the start of a sequence of LiteVector elements (the start of a message or beginning of a file for example).

Individual elements are unaligned. It is good form for vectors to be aligned to the size of their type. For example, a vector of 32-bit float values would have the first byte of the first element start at a 4-byte offset from the start of the message. This can be accomplished by inserting `NOP` values in the stream before the vector's tag.

Aligning vectors allows deserializer code to optimize vector processing by processing vectors in place. If they are not aligned, then the receiver may need to perform additional work in order to use the vector data. However, serialization, deserialization, and alignment needs can change between applications, so alignment is not strictly required. Standard conformant deserializers should be able to deal with both aligned and unaligned vectors.

## Standalone Integer Sizing
For arrays of integers, LiteVectors takes the relatively hands off approach of 'set a datatype and go to town' - all elements are the same size and type.

For standalone integers, however, different environments use different strategies. Some like C allow you to define an 'int' that adapts to the native word size of the hardware. Some JavaScript engines track whether a 'number' is an integer or not, and can fit it into about 2^53. Python 3 implements arbitrarily large integers.

In order to implement a normalized interoperable format and not be wasteful with space, LiteVectors recommends a Goldilocks 'best fit' integer encoding for standalone integers. The rules are simple:

- Non-negative integers are encoded as unsigned types
- Negative integers are encoded as signed types
- Integers are encoded in the smallest type that will hold them (among u8, u16, u32, u64, i8, i16, i32, i64).

By following these rules, most integers are stored relatively efficiently. Additionally, integer format is deterministic even between platforms with normally incompatible integer representations.

Encoders should goldilocks fit standalone integers, and decoders should be ready to deserialize them into whichever platform type is appropriate.

## JSON Representation
LiteVectors support JSON representation for interoperability and diagnostics - JSON is often easier to read than hex.

| LiteVector     | JSON   | JSON Example                | Note                                                                                  |
| :------------- | :----- | :-------------------------- | :------------------------------------------------------------------------------------ |
| nil            | null   |                             | JSON null                                                                             |
| struct         | object | { 'age': 2, 'cat': true }   | JSON objects                                                                          |
| list           | array  | [1, "brown", "cat"]         | JSON array                                                                            |
| string         | string | "Hello JSON                 |                                                                                       |
| bool           | bool   | true, false                 |                                                                                       |
| (i,u)(8,16,32) | number | -100, 0, 200                | Integers are JSON numbers                                                             |
| i64, u64       | string | "-100", "0", "200"          | JSON string to avoid number overflow                                                  |
| f32, f64       | number | 1.2, -5.9, "NaN, "Infinity" | JSON value is a number or one of the special strings "NaN", "Infinity" or "-Infinity" |
| vector         | array  | [1, 2, 3, 4]                | Vectors of primitives follow the same encoding above, but in a JSON array             |
