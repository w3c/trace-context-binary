# Rationale for decision on binary format

Binary format is similar to proto encoding without any reference on protobuf
project. It uses field identifiers in bytes in front of field values.

## Field identifiers

Protocol uses field identifiers for fields like `trace-id`, `parent-id`,
`trace-flags` and tracestate entries. The purpose of the field identifiers is
two-fold. First, allow to remove existing fields or add new ones going forward.
Second, provides an additional layer of validation of the format.

## How can we add new fields

If we follow the rules that we always append the new ids at the end of the
buffer we can add up to 127. After that we can either use varint encoding or
just reserve 255 as a continuation byte. Assumption at the moment is that
specification will never get to this point.

## Why custom binary protocol

We didn't find non-proprietary wide used binary protocol that can be used in
this specification.
