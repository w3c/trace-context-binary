# Binary format

Binary format document describes how to encode each field - `traceparent` and
`tracestate`. The binary format should be used to encode the values of these
fields. This specification does not specify how these fields should be stored
and sent as a part of a binary payload. The basic implementation may serialize
those as size of the field followed by the value.

Specification operates with bytes - unsigned 8-bit integer values representing
values from `0` to `255`. Byte representation as a set of bits (big or little
endian) MUST be defined by underlying platform and out of scope of this
specification.

## `Traceparent` binary format

The field `traceparent` encodes the version of the protocol and fields
`trace-id`, `parent-id` and `trace-flags`. Each field starts with the one byte
field identifier with the field value following immediately after it. Field
identifiers are used as markers for additional verification of the value
consistency and may be used in future for the versioning of the `traceparent`
field.

``` abnf
traceparent     = version version_format  
version         = 1BYTE                   ; version is 0 in the current spec
version_format  = "{ 0x0 }" trace-id "{ 0x1 }" parent-id "{ 0x2 }" trace-flags
trace-id        = 16BYTES
parent-id       = 8BYTES
trace-flags     = 1BYTE  ; only the least significant bit is used
```

Unknown field identifier (anything beyond `0`, `1` and `2`) should be treated as
invalid `traceparent`. All zeroes in `trace-id` and `parent-id` invalidates the
`traceparent` as well.

## Serialization of `traceparent`

Implementation MUST serialize fields into the field ordering sequence. In other
words, `trace-id` field should be serialized first, `parent-id` second and
`trace-flags` - third.

Field identifiers should be treated as unsigned byte numbers and should be
encoded in big-endian bit order.

Fields `trace-id` and `parent-id` are defined as a byte arrays, NOT a long
numbers. First element of an array MUST be copied first. When array is
represented as a memory block of 16 bytes - serialization of `trace-id` would be
identical to `memcpy` method call on that memory block. This may be a concern
for implementations casting these fields to integers - protocol is NOT defining
whether those byte arrays are ordered as big endian or little endian and have a
sign bit.

If padding of the field is required (`traceparent` needs to be serialized into
the bigger buffer) - any number of bytes can be appended to the end of the
serialized value.

## `traceparent` example

``` js
{0,
  0, 75, 249, 47, 53, 119, 179, 77, 166, 163, 206, 146, 157, 0, 14, 71, 54,
  1, 52, 240, 103, 170, 11, 169, 2, 183,
  2, 1}
```

This corresponds to:

- `trace-id` is `{75, 249, 47, 53, 119, 179, 77, 166, 163, 206, 146, 157, 0, 14,
  71, 54}` or `4bf92f3577b34da6a3ce929d000e4736`.
- `parent-id` is `{52, 240, 103, 170, 11, 169, 2, 183}` or `34f067aa0ba902b7`.
- `trace-flags` is `1` with the meaning `recorded` is true.

## `tracestate` binary format

List of up to 32 name-value pairs. Each list member starts with the 1 byte field
identifier `0`. The format of list member is a single byte key length followed
by the key value and single byte value length followed by the encoded value.
Note, single byte length field allows keys and values up to 256 bytes long. This
limit is defined by [trace
context](https://w3c.github.io/trace-context/#header-value) specification.
Strings are transmitted in ASCII encoding.

``` abnf
tracestate      = list-member 0*31( list-member )
list-member     = "0" key-len key value-len value
key-len         = 1BYTE ; length of the key string
value-len       = 1BYTE ; length of the value string
```

Zero length key (`key-len == 0`) indicates the end of the `tracestate`. So when
`tracestate` should be serialized into the buffer that is longer than it
requires - `{ 0, 0 }` (field id `0` and key-len `0`) will indicate the end of
the `tracestate`.

## `tracestate` example

``` js
{ 0,  3,  102, 111, 111,  16,  51, 52, 102, 48, 54, 55, 97, 97, 48, 98, 97, 57, 48, 50, 98, 55,
  0,  3,   98,  97, 114,  4,   48, 46, 50, 53,  }

```

This corresponds to 2 tracestate entries:

`foo=34f067aa0ba902b7,bar=0.25`
