LiteVectors Specification
===========




## Data Model
The data model is most similar to JSON with three high level structures:

* struct - name/value pairs (also known as a dictionary, object, map, or associative array in other some environments)
* list - an ordered sequence of values (a generic array)
* vector - a length prefixed dense array of values (a strongly typed array)

## Primitives
There are 16 primitive types:

| Type Code | Name   | Description                                    |
| --------: | :----- | :--------------------------------------------- |
|         0 | nil    | Null / No value                                |
|         1 | struct | Start of an `string`/value associative map     |
|         2 | list   | Start of a (heterogenous) sequence of elements |
|         3 | end    | End tag for a `struct` or `list`               |
|         4 | string | UTF-8 encoded bytes                            |
|         5 | bool   | True / False                                   |
|         6 | u8     | Unsigned 8-bit integer                         |
|         7 | u16    | Unsigned 16-bit integer                        |
|         8 | u32    | Unsigned 32-bit integer                        |
|         9 | u64    | Unsigned 64-bit integer                        |
|        10 | i8     | Signed 8-bit integer                           |
|        11 | i16    | Signed 16-bit integer                          |
|        12 | i32    | Signed 32-bit integer                          |
|        13 | i64    | Signed 64-bit integer                          |
|        14 | f32    | IEEE-754 32-bit floating point number          |
|        15 | f64    | IEEE-754 64-bit floating point number          |

## Wire Format

LiteVector elements follow a _tag, length, value_ convention - all elements are preceded with a one byte tag, and either a fixed length value, or length field and vector of values. 

```
┌─────┐
│ Tag │
└─────┘
┌─────┬────────┐
│ Tag │ Value  │
└─────┴────────┘
┌─────┬────────┬────────┐
│ Tag │ Length │ Vector │
└─────┴────────┴────────┘
```
## Tag Format
A tag byte is divided into two fields: a type code and a size code. The type code is stored in the top 4 bits, the size code is stored in the bottom 4 bits:
```
     ┌────────┬────────┐
     │  Type  │  Size  │
     └────────┴────────┘
Bits: 7 6 5 4 | 3 2 1 0
```

### Type Codes
| Type Code | Name   | Type Size |                  Min |                  Max |
| --------: | :----- | --------: | -------------------: | -------------------: |
|         0 | nil    |         0 |                      |                      |
|         1 | struct |         0 |                      |                      |
|         2 | list   |         0 |                      |                      |
|         3 | end    |         0 |                      |                      |
|         4 | string |         1 |                      |                      |
|         5 | bool   |         1 |            0 (false) |           255 (true) |
|         6 | u8     |         1 |                    0 |                  255 |
|         7 | u16    |         2 |                    0 |                65535 |
|         8 | u32    |         4 |                    0 |           4294967295 |
|         9 | u64    |         8 |                    0 | 18446744073709551615 |
|        10 | i8     |         1 |                 -128 |                  127 |
|        11 | i16    |         2 |               -32768 |                32767 |
|        12 | i32    |         4 |          -2147483648 |           2147483647 |
|        13 | i64    |         8 | -9223372036854775808 |  9223372036854775807 |
|        14 | f32    |         4 |         -3.40282e+38 |          3.40282e+38 |
|        15 | f64    |         8 |        -1.79769e+308 |         1.79769e+308 |

### Size Codes
| Size Code | Name   | Description                              |
| :-------- | :----- | :--------------------------------------- |
| 0         | single | Single element, inline                   |
| 1         | size_1 | Vector with 1 byte (uint8) length field  |
| 2         | size_2 | Vector with 2 byte (uint16) length field |
| 3         | size_4 | Vector with 4 byte (uint32) length field |
| 4         | size_8 | Vector with 8 byte (uint64) length field |
| 5-15      | unused |                                          |

The tag value `0xFF` is reserved as a NOP element, and should be ignored by a decoder.

Any other tag with a size code that is not `single` is followed by an unsigned integer containing the length of the following vector. Lengths are always in bytes (not number of elements). 

## Types
* All numeric values are _little endian_
* Signed integers are encoded in _two's complement_
* `bool` is stored as a one byte value, 0 false, non-zero true
* `float32` is the IEEE 754 _binary32_ encoding
* `float64` is the IEEE 754 _binary64_ encoding
* `string` is UTF-8

## NOP tags and Alignment
Alignment refers to the position of an element with respect to the start of a sequence of LiteVector elements (the start of a message or beginning of a file for example).

Individual elements are unaligned. It is good form for vectors to be aligned to the size of their type. For example, a vector of 32-bit float values would have the first byte of the first element start at a 4-byte offset from the start of the message. This can be accomplished by inserting `nop` elements in the stream before the vector's tag.

Aligning vectors allows deserializer code to optimize vector processing by processing vectors in place. If they are not aligned, then the receiver may need to perform additional work in order to use the vector data. However, serialization, deserialization, and alignment needs can change between applications, so alignment is not strictly required. Standards conformant deserializers should be able to deal with both aligned and unaligned vectors.

## Standalone Integer Sizing
Standalone integers (those that are not in a vector of a fixed type) may be stored in Goldilocks encoding to save space. Goldilocks encoding is simply using a 'best fit' type for the value

     - Positive integers should are stored as unsigned
     - Negative integers should are stored as signed
     - Values are encoded using the smallest type size that fits their quantity

Vector length fields should be goldilocks encoded to accommodate the required length value.

Goldilocks encoding makes LiteVector representations consistent between environments that may not have fixed sized native integers such as Python or JavaScript.

## Field Ordering within Structs
LiteVectors requires fields in a `struct` to be written in a consistent order. The rational for this is that fields present earlier in a `struct` may affect deserialization of fields present later. Middleware applications processing and forwarding LiteVector messages must take care to maintain field order for processed messages.

# Limits and Validation

The NOP tag `0xFF` should be discarded before any other processing. 

A `SizeCode` > 4 is invalid and must be rejected.
A `string` must be valid UTF-8. The `string` datatype has a unit length of 1 byte, so a string with a `SizeCode` of 0 must be an ASCII character <= 0x7F.

The zero length elements `nil`, `struct`, `list`, and `end` must have a `SizeCode` of 0.

All length fields are expressed in bytes. A vector length must always be a multiple of its type length. Primitive type sizes greater than one are always a power of two, so this can be efficiently checked with a statement similar to:

``` c
if ( vectorLength & (typeSize-1)) != 0) {
     /* This is an error */
}
```


All bit patterns for IEEE-754 floating point numbers are valid. General purpose serializers should _not_ scrub or alter floating point values, but pass them intact to application code to handle special cases.

All `struct` field names are `string`s.

Individual applications and libraries may set their own limits (or make them configurable) for the following:
- Vector size limits
- Struct and list nesting depth
- Number of consecutive `nop` bytes permitted

## Stream Semantics
Within a given context such as a stream, socket, file, etc, LiteVector semantics are to behave as if in a _list_. Standalone elements are allowed.

## JSON Encoding
LiteVectors support JSON encoding for interoperability and diagnostics (JSON is usually easier to read than hex).


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