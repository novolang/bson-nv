# Changelog

All notable changes to bson-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1]

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `bsonvalue` — the load-bearing interface. `BsonDoc` is a LIST of
  elements and not a map, because two facts of the format forbid a map:
  BSON preserves the order elements were written in, so a driver that
  reordered them would change the bytes a checksum was taken over, and
  BSON permits the same key twice, so a validator has to be able to see
  both elements. `get` answers the first element with a key and
  `count_of` is what finds the duplicate. `BsonValue` has an arm for
  every one of the twenty-one type codes BSON 1.1 assigns, the five
  deprecated ones included.
- `bsonwrite` — the asymmetry that is this package's other decision:
  **every type reads, only the current types write**. A document
  carrying an undefined, a dbPointer, a symbol, javascriptWithScope or
  a binary element of subtype `0x02` or `0x03` is refused with
  `BsonDeprecatedOnWrite` naming the key. Writing them would let a
  program produce new documents in a form with no future; replacing
  them silently would change a document's meaning under a caller who
  asked for a round trip. `modernised` performs the substitution
  deliberately, and it is separate because one of its cases REMOVES an
  element.
- `bsonread` — one buffer, one document, because BSON is length-prefixed
  from the outside in. `document_length` is the half a stream reader
  needs, `read_prefix` and `read_all` are for concatenated documents,
  and `value_span` answers a key's byte range without building the
  tree — the only way to get a sub-document's exact original bytes back,
  since a round trip through the writer is not guaranteed to reproduce
  them.
- `bsontype` — the scalars with rules of their own. `object_id_from_parts`
  is the constructor that keeps this package `core`: generating an
  ObjectId needs a clock and a random source, so the caller reads both
  and hands the parts over. `BsonDecimal128` is the sixteen bytes and
  nothing else — carried and rendered, never computed with.
- `bsonjson` — Extended JSON version 2 against `std.json`'s own value
  type rather than a JSON model of this package's own. The mode is a
  required argument because the canonical form round-trips and the
  relaxed form does not, and reading accepts both because the mode is a
  writing choice.
- `bsonerror` — twenty reasons, each carrying the byte offset it was
  found at, with `is_document_fault` separating "these bytes are
  corrupt" from "this request asked for the wrong thing".

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  bson-nv.<module>.<fn>`.
- **The BSON corpus is named but not generated.** The suite carries the
  cases from bsonspec.org and the shape of the corpus's own triples; the
  generated run over the whole corpus lands with the implementation.
- **No `tests/embedded_probe.nv`.** The absence is a claim not made
  rather than a claim skipped: a document is one allocation per element,
  and the Extended JSON conversion speaks `std.json`, which refuses the
  embedded tier outright.
