# Test Vectors
Test vectors are like unit tests for a protocol - and perhaps one of the more important tools for creating interoperable systems.

The LiteVector test suite consists of both _positive_ test vectors - examples of encoded data that follow the protocol, and _negative_ test vectors - examples of data that violate the protocol (and should raise an error in a decoder). 

Like yin and yang define themselves in terms of each other, positive and negative test vectors provide the benefit of outlining what is and what is _not_ valid under the specification, and help highlight inconsistencies and misinterpretations.

The test vectors are text files with alternating lines of test description and hex encoded data. For example, here are two positive vectors:

```
struct: {'a': {'b': {'c': 5} } }
1040611040621040636005303030
u8[]: [1, 2, 3, 4]
610401020304
```

An implementation wanting to test against these would keep the first line as a description (`struct: {'a': {'b': {'c': 5} } }`) for printing or feedback, and then parse the second line `1040611040621040636005303030` into binary to feed to a parser. The parser should successfully read and validate this data. The next line contains another test description followed by another test vector.

The current set of _positive_ test vectors: [litevectors_positive.txt](https://raw.githubusercontent.com/LitePipe/ltvgo/main/testvectors/litevectors_positive.txt).

The current set of _negative_ test vectors are [litevectors_negative.txt](https://raw.githubusercontent.com/LitePipe/ltvgo/main/testvectors/litevectors_negative.txt).

If you encounter an error that you think could have been caught with a good test vector, submit an [issue](https://github.com/LitePipe/litevectors.org/issues) - and let's make everybody's parsers better.
