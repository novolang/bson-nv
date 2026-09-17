# bson-nv

BSON is a binary serialisation format for documents, specified at
[bsonspec.org](https://bsonspec.org/spec.html) as version 1.1. It is
what MongoDB stores on disk and what its drivers put on the wire. This
package reads and writes BSON documents in novo-lang, and converts them
to and from MongoDB's Extended JSON.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What BSON is

A BSON **document** is a four-byte little-endian length, then zero or
more elements, then a `0x00` terminator. The length counts itself and
the terminator, so the smallest document is the five bytes
`05 00 00 00 00`.

An **element** is a one-byte **type code**, a NUL-terminated key, and a
payload whose shape the type code fixes. An element carrying a string
writes a four-byte length, the bytes, and a NUL; the length counts the
NUL. An element carrying an embedded document writes that document
whole, length prefix and all.

An **array** is stored as a document whose keys are the decimal strings
`"0"`, `"1"`, `"2"` and so on, in order. There is no separate array
encoding. A three-element array therefore costs more bytes than the same
array in JSON.

BSON 1.1 assigns twenty-one type codes.

| Code | Type | Payload |
| --- | --- | --- |
| `0x01` | double | 8 bytes, IEEE 754 binary64 |
| `0x02` | string | length, UTF-8 bytes, NUL |
| `0x03` | document | a document |
| `0x04` | array | a document with decimal string keys |
| `0x05` | binData | length, subtype byte, bytes |
| `0x06` | undefined | nothing. Deprecated |
| `0x07` | objectId | 12 bytes |
| `0x08` | bool | 1 byte, `0x00` or `0x01` |
| `0x09` | date | 8 bytes, signed milliseconds since the Unix epoch |
| `0x0A` | null | nothing |
| `0x0B` | regex | two NUL-terminated strings |
| `0x0C` | dbPointer | a string and 12 bytes. Deprecated |
| `0x0D` | javascript | a string |
| `0x0E` | symbol | a string. Deprecated |
| `0x0F` | javascriptWithScope | a length, a string, a document. Deprecated |
| `0x10` | int | 4 bytes, signed |
| `0x11` | timestamp | 8 bytes, an internal replication value |
| `0x12` | long | 8 bytes, signed |
| `0x13` | decimal | 16 bytes, IEEE 754-2008 decimal128 |
| `0xFF` | minKey | nothing. Compares below every value |
| `0x7F` | maxKey | nothing. Compares above every value |

A **binary element** carries a subtype byte before its bytes, so that a
UUID and a checksum stored in the same field are told apart.

| Subtype | Meaning |
| --- | --- |
| `0x00` | generic |
| `0x01` | function |
| `0x02` | binary, old — a second length prefix inside the payload. Deprecated |
| `0x03` | UUID, old — a byte order that differed between drivers. Deprecated |
| `0x04` | UUID, RFC 4122 byte order |
| `0x05` | MD5 digest |
| `0x06` | encrypted payload |
| `0x07` | compressed column |
| `0x08` | sensitive value |
| `0x09`–`0x7F` | reserved. Refused |
| `0x80`–`0xFF` | user-defined |

**Extended JSON** is MongoDB's way of writing a BSON document as JSON
without losing the types. JSON has six kinds of value and BSON has
twenty-one, so a plain JSON rendering turns an int32, an int64, a double
and a decimal128 all into a number. Extended JSON version 2 gives each
type a reserved member name beginning with `$`, so
`{"$numberLong": "9007199254740993"}` is an int64 and survives a parser
that would have rounded the number. It has two forms: **canonical**,
which wraps every type, and **relaxed**, which writes numbers and dates
plainly so a person can read them.

## Install

```
novo pkg add bson-nv
```

## Example

```novo
use std.bytes
use bsonread
use bsonvalue
use bsonjson

fn main() [io]
    // The bytes of {"hello": "world"}, as bsonspec.org writes them.
    let raw = bytes.from_hex("160000000268656c6c6f0006000000776f726c640000")
              ?? bytes.zeros(0)

    match bsonread.read(raw)
        Err(e) => println("not a document: ${e.message()}")
        Ok(doc) =>
            // Look one key up. The first element with that key wins.
            match bsonvalue.get(doc, "hello")
                Err(e) => println("no such key: ${e.message()}")
                Ok(v)  =>
                    // Ask for the type the element actually has.
                    match bsonvalue.as_str(v)
                        Err(e) => println("not a string: ${e.message()}")
                        Ok(s)  => println(s)

            // The same document as Extended JSON a person can read.
            println(bsonjson.doc_to_text(doc, BsonRelaxedExt))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: bson-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bsonerror` | Every refusal the reader, the writer and the JSON conversion can produce, each with a byte offset. |
| `bsontype` | The scalar types with rules of their own: the ObjectId, the binary subtype table, the regular expression, the replication timestamp and the decimal128. |
| `bsonvalue` | The twenty-one element types, the document that keeps its order, and the accessors over both. |
| `bsonread` | Bytes to a document, with the limits a reader is configured with and a scan that finds one value without building the rest. |
| `bsonwrite` | A document to bytes, and the refusal of the deprecated types. |
| `bsonjson` | Extended JSON version 2, both ways, against the standard library's JSON values. |

## How to choose an entry point

**`bsonread.read` takes one buffer and gives one document.** Use it
when the bytes are exactly one document and nothing else. It refuses a
buffer with anything after the document, so a second document is never
silently dropped.

**`bsonread.read_prefix` takes one document off the front** and says how
many bytes it used. Use it when several documents are concatenated, as
they are in a mongodump archive and an oplog dump. `read_all` is the
same walk to the end of the buffer.

**`bsonread.document_length` reads the four-byte prefix alone.** Use it
when the bytes arrive over a socket: take four bytes, learn the length,
wait for the rest, then call `read`.

**`bsonread.value_span` finds one key's value without building the
document.** Use it to pull an `_id` out of a large stored document, and
to get the exact original bytes of a sub-document, which reading and
writing back does not guarantee.

**`bsonread.validate` walks the structure and builds nothing.** Use it
when the only question is whether the bytes are sound.

**`bsonwrite.can_write` asks whether the writer will take a document**
before any bytes are produced. Use it when filtering a stream of
documents that may have come from an old driver.

## The rules a user needs

1. **A document is an ordered list of elements, not a map.** BSON
   preserves the order elements were written in, and a driver that
   reordered them would change the bytes a checksum was taken over.
   `bsonvalue.BsonDoc` is a list, and nothing in this package sorts it.
2. **A document may carry the same key twice.** The server rejects such
   a document; a file on disk may still hold one.
   `bsonvalue.get` answers the first element with that key, and
   `bsonvalue.count_of` is the only way to find out there was a second.
3. **Every type reads. Only the current types write.** BSON 1.1
   deprecates undefined, dbPointer, symbol, javascriptWithScope, and
   binary subtypes `0x02` and `0x03`. All six are read into their own
   values. `bsonwrite.write` answers `BsonDeprecatedOnWrite` naming the
   key it found one under.
4. **Bytes round-trip in one direction only.** Every document this
   package writes reads back identical. A document read from a legacy
   source may not be writable at all.
   `bsonvalue.first_deprecated` answers that before the writer does.
5. **`bsonwrite.modernised` is the deliberate conversion.** A symbol
   becomes a string, javascriptWithScope becomes a document with
   `$code` and `$scope`, a dbPointer becomes a document with `$ref` and
   `$id`, and an undefined element is **removed**. Removing an element
   changes a document's shape, which is why it is a separate call and
   not something `write` does quietly.
6. **An int32 and an int64 are different types and stay different.**
   `bsonvalue.as_int` answers for both, and the element keeps the width
   it was read with, because widening every integer would change the
   bytes of every document.
7. **A string element's length prefix counts its trailing NUL.**
   A key does not have one: a key is a C string with no prefix.
8. **Neither a key nor a regex pattern may contain a `0x00`.** Both are
   C strings on the wire. The reader answers `BsonEmbeddedNul` and the
   writer refuses.
9. **An array's keys must be `"0"`, `"1"`, `"2"` and so on, in order.**
   A `0x04` element whose keys are anything else is `BsonBadArrayKey`.
10. **Binary subtypes `0x09` to `0x7F` are refused.** BSON 1.1 neither
    assigns them nor opens them to users, so accepting one is accepting
    a meaning a later version of the format will define.
11. **A UTC datetime is signed milliseconds since the Unix epoch**, so a
    date before 1970 is negative. It is a number here, not a date: this
    package reads no clock and turning milliseconds into a civil date is
    another package's work.
12. **A replication timestamp is not a date.** Type `0x11` is an
    internal value the server uses for ordering in the oplog. A date a
    user reads is type `0x09`.
13. **An ObjectId is assembled from parts the caller supplies.**
    `bsontype.object_id_from_parts` takes the four-byte timestamp, five
    random bytes and a three-byte counter, because generating one needs
    a clock and a random source and this package has neither.
14. **A decimal128 is carried, not computed.** The sixteen bytes go in
    and come out unchanged, and the string form is the one the Extended
    JSON specification fixes. There is no arithmetic.
15. **Extended JSON's mode is an argument with no default.** The
    canonical form round-trips and the relaxed form does not:
    `{"n": 1}` read back from relaxed is an int32 whatever it started
    as. `bsonjson.relaxed_round_trips` says which values survive.
16. **Reading Extended JSON accepts both forms.** The mode is a writing
    choice only, because a relaxed document may still carry a
    `$numberDecimal` and the specification's parsing rules say to accept
    either.
17. **A plain JSON number takes the narrowest type that holds it**: an
    int32 if it fits thirty-two bits, an int64 if it fits sixty-four, a
    double if it has a fraction or an exponent, and
    `BsonUnrepresentableJson` otherwise.
18. **`{"$oid": "…"}` is an ObjectId and `{"$oid": "…", "x": 1}` is
    not.** A type wrapper is an object whose members are only the
    reserved names for one type. `bsonjson.is_type_wrapper` is the
    function that decides.

## What is not included

- **A MongoDB client.** Opening a connection costs `[net]` and this
  package declares no effects. This is the document format the wire
  protocol carries, and a client is built on top of it.
- **The MongoDB wire protocol.** `OP_MSG`, its sections and its
  checksum are a separate format whose payloads are the documents this
  package reads.
- **Decimal arithmetic.** A decimal128 element is its sixteen bytes and
  its string form. Adding two of them belongs to a decimal package.
- **A clock and a random source.** See rules 11 and 13.
- **Comparison in the server's sort order.** MongoDB orders values
  across types by a bracket table of its own, and a program that sorts
  by it is reproducing a server behaviour rather than reading a format.
- **Evaluation of a `javascript` element.** The code is text here and
  nothing runs it.
- **JSON Schema validation.** The server's document validators are a
  query-language feature, not part of the format.

## Related packages

- [cbor-nv](https://novo-lang.org/packages/cbor-nv) is CBOR, RFC 8949: a
  binary format with the same job and a smaller encoding, and with
  tags in place of a fixed type table. Take it when both ends are
  yours. Take this package when one end is MongoDB.
- [msgpack-nv](https://novo-lang.org/packages/msgpack-nv) is MessagePack,
  which is smaller again and has no document length prefix, so it cannot
  be skipped over without being parsed.
- [json-nv and the standard library's `std.json`](https://novo-lang.org/packages/jsonpath-nv)
  hold the JSON values this package's Extended JSON conversion produces
  and consumes.
- [decimal-nv](https://novo-lang.org/packages/decimal-nv) is decimal
  arithmetic. Convert a `BsonDecimal128` through its string form to
  compute with it.
- [calendar-nv](https://novo-lang.org/packages/calendar-nv) turns the
  milliseconds a date element holds into a civil date.

## Test vectors

The normative sources are the BSON 1.1 specification at bsonspec.org for
the grammar and the type table, the MongoDB ObjectId specification for
the twelve-byte layout, IEEE 754-2008 for decimal128, and the MongoDB
Extended JSON version 2 specification for the wrappers and the two
forms.

The vectors are the **BSON corpus** published in the MongoDB
specifications repository: a set of files each holding
`canonical_bson` / `canonical_extjson` / `relaxed_extjson` triples, plus
lists of documents that must be refused at decode and Extended JSON that
must be refused at parse. It is the suite every driver is measured
against, and the whole of it becomes a generated run beside the hand
suite when the implementation lands.

```bash
novo test tests/bson_tests.nv    # the grammar, the types, the two conversions
```

The suite asserts that the empty document is five bytes, that a
length prefix which disagrees with the buffer is refused, that a second
document in the buffer is not ignored, that an int32 keeps its width,
that a duplicate key stays visible, that an array is a document with
decimal string keys, that the writer refuses a symbol and names its key,
that the reserved binary subtypes are refused, that regex options must
be a sorted subset of `ilmsux`, and that the canonical and relaxed forms
of an int64 differ.

The tests compile today and fail at run, each on the
`not implemented: bson-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bsonvalue.BsonValue`, `.BsonElem`, `.BsonDoc` and the other types | the types are declared |
| `bsonerror.offset_of`, `.is_document_fault`, `.code_of`, `BsonError.message` | no |
| `bsontype.object_id`, `.object_id_from_hex`, `.object_id_from_parts` and the ObjectId readers | no |
| `bsontype.binary`, `.binary_check`, `.subtype_code`, `.subtype_of_code`, `.subtype_name`, `.is_deprecated_subtype` | no |
| `bsontype.regex`, `.regex_check`, `.timestamp`, `.timestamp_u64`, `.db_pointer` | no |
| `bsontype.decimal128` and the five readers over it | no |
| `bsonvalue.type_code`, `.type_name`, `.type_of_code`, `.is_deprecated`, `.first_deprecated` | no |
| `bsonvalue.as_float`, `.as_int`, `.as_str`, `.as_bool`, `.as_binary`, `.as_doc`, `.as_array`, `.as_object_id`, `.as_datetime_millis`, `.as_decimal128` | no |
| `bsonvalue.doc`, `.doc_of`, `.elem`, `.append`, `.len`, `.at`, `.keys` | no |
| `bsonvalue.get`, `.lookup`, `.has`, `.count_of`, `.replace`, `.remove`, `.path` | no |
| `bsonvalue.array_doc`, `.is_array_doc` | no |
| `bsonread.default_limits`, `.document_length`, `.read`, `.read_with` | no |
| `bsonread.read_prefix`, `.read_prefix_with`, `.read_all`, `.validate`, `.value_span` | no |
| `bsonwrite.encoded_len`, `.write`, `.write_elem`, `.write_all`, `.can_write`, `.modernised` | no |
| `bsonjson.mode_name`, `.doc_to_json`, `.value_to_json`, `.doc_to_text` | no |
| `bsonjson.doc_from_json`, `.value_from_json`, `.doc_from_text` | no |
| `bsonjson.is_type_wrapper`, `.wrapper_name`, `.relaxed_round_trips` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
