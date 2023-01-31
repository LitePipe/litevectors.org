# Specification

LiteVectors is a binary format 

The data model is most similar to JSON, with primitive types like strings and numbers, along with a small handful of organizational structures:

* struct - name/value pairs (also known as a dictionary, object, map, or associative array in some environments)
* list - an ordered sequence of values (a generic array)
* vector - a length prefixed dense array of values (a strongly typed array)

## Element Layout

LiteVector elements follow a traditional _tag, length, value_ convention - all elements consist of a one byte tag and either a fixed length value, or length field and vector of values:

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
A tag byte is divided into two fields: a `type code` and a `size code`. The `type code` is stored in the high order 4 bits while the `size code` is stored in the lower 4 bits:

```
     ┌────────┬────────┐
     │  Type  │  Size  │
     └────────┴────────┘
Bits: 7 6 5 4 | 3 2 1 0
```

The tag value `0xFF` is reserved as a NOP element, and should be ignored by a decoder. The NOP tag is used in advanced scenarios where an implementation needs precise control over vector alignment.

## Primitive Types
There are 16 primitive types. Each type has a 'type size' - the size of a value of that type. 

* All numeric values are _little endian_
* Signed integers are encoded in _two's complement_
* `bool` is stored as a one byte value, 0 false, non-zero true
* `float32` is the IEEE 754 _binary32_ encoding
* `float64` is the IEEE 754 _binary64_ encoding
* `string` is UTF-8


| Type Code | Name   | Description                                        | Type Size |                  Min |                  Max |
| --------: | :----- | :------------------------------------------------- | :-------: | -------------------: | -------------------: |
|         0 | nil    | Null / No value                                    |     0     |                      |                      |
|         1 | struct | Beginning of a `string`/value associative map      |     0     |                      |                      |
|         2 | list   | Beginning of a (heterogenous) sequence of elements |     0     |                      |                      |
|         3 | end    | End tag for a `struct` or `list`                   |     0     |                      |                      |
|         4 | string | UTF-8 encoded bytes                                |     1     |                      |                      |
|         5 | bool   | True / False                                       |     1     |            0 (false) |           255 (true) |
|         6 | u8     | Unsigned 8-bit integer                             |     1     |                    0 |                  255 |
|         7 | u16    | Unsigned 16-bit integer                            |     2     |                    0 |                65535 |
|         8 | u32    | Unsigned 32-bit integer                            |     4     |                    0 |           4294967295 |
|         9 | u64    | Unsigned 64-bit integer                            |     8     |                    0 | 18446744073709551615 |
|        10 | i8     | Signed 8-bit integer                               |     1     |                 -128 |                  127 |
|        11 | i16    | Signed 16-bit integer                              |     2     |               -32768 |                32767 |
|        12 | i32    | Signed 32-bit integer                              |     4     |          -2147483648 |           2147483647 |
|        13 | i64    | Signed 64-bit integer                              |     8     | -9223372036854775808 |  9223372036854775807 |
|        14 | f32    | IEEE-754 32-bit floating point number              |     4     |         -3.40282e+38 |          3.40282e+38 |
|        15 | f64    | IEEE-754 64-bit floating point number              |     8     |        -1.79769e+308 |         1.79769e+308 |


## Size Codes
There are 5 size codes, indicating the size of the length field following the tag byte. In the case of a single element, the 0 `size code` indicates that no length field follows the tag. Any tag with a `size code` that is not 0 is followed by an unsigned integer containing the length of the following vector. The length field is always in bytes (not number of elements).

| Size Code | Name   | Description                              |
| :-------- | :----- | :--------------------------------------- |
| 0         | single | Single element, inline                   |
| 1         | size_1 | Vector with 1 byte (uint8) length field  |
| 2         | size_2 | Vector with 2 byte (uint16) length field |
| 3         | size_4 | Vector with 4 byte (uint32) length field |
| 4         | size_8 | Vector with 8 byte (uint64) length field |
| 5-15      | unused |                                          |


## Field Ordering within Structs
LiteVectors requires fields in a `struct` to be written in a consistent order. The rational for this is that fields present earlier in a `struct` may affect deserialization of fields present later. Individual applications may selectively enforce or relax this requirement as they desire, but middleware services and libraries processing and forwarding LiteVector messages must take care to maintain the transmitted field order.

## Default List Semantics
Within a given context such as a stream, socket, file, etc, LiteVector processing semantics are to behave as if in a _list_. Standalone elements are allowed, and multiple elements may be concatenated one after another.

By convention, a file containing raw LiteVector elements should have a `.ltv` extension.

## Limits and Validation
The NOP tag `0xFF` should be discarded before any other processing. 

A `size code` > 4 is invalid and must be rejected.
A `string` must be valid UTF-8. The `string` datatype has a unit length of 1 byte, so a string with a `size code` of 0 must be an ASCII character <= 0x7F.

The zero length elements `nil`, `struct`, `list`, and `end` must have a `size code` of 0.

All length fields are expressed in bytes. A vector length must be a multiple of its type length. Primitive type sizes greater than one are always a power of two, so this can be efficiently checked with a statement similar to:

``` c
if ( vectorLength & (typeSize-1)) != 0) {
     /* This is an error */
}
```

All bit patterns for IEEE-754 _binary32_ and _binary64_ floating point numbers are valid. General purpose serializers should _not_ scrub or alter floating point values, but pass them intact to application code to handle special cases.

All `struct` field names are strings.

Individual applications and libraries may set their own limits (or make them configurable) for the following:

- Vector size limits
- Struct and list nesting depth
- Number of consecutive `nop` bytes permitted
