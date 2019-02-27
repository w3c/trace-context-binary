# De-serialization algorithms

This is non-normative section that describe de-serialization algorithm
that may be used to parse `traceparent` and `tracestate` field values.

## De-serialization of `traceparent`

Let's assume the algorithm takes a buffer - bytes array - and can set
and shift cursor in the buffer as well as validate whether the end of
the buffer was reached or will be reached after reading the given number
of bytes. This algorithm can work on stream of bytes. De-serialization
of `traceparent` MAY be done in the following sequence:

1. If buffer is empty - RETURN invalid status `BUFFER_EMPTY`. Set a cursor to
   the first byte.
2. Read the `version` byte at the cursor position. Shift cursor to `1` byte.
3. If at the end of the buffer RETURN invalid status `TRACEPARENT_INCOMPLETE`.
4. **Parse `trace-id`**. Read the field identifier byte at the cursor
   position. If NOT `0` - go to step `8. Report invalid field`.
   Otherwise - check that remaining buffer size is more or equal to `16`
   bytes. If shorter - RETURN invalid status `TRACE_ID_TOO_SHORT`.
   Otherwise read the next `16` bytes for `trace-id` and shift cursor to
   the end of those `16` bytes.
5. **Parse `trace-id`**. Read the field identifier byte at the cursor
   position. If NOT `1` - go to step `8. Report invalid field`.
   Otherwise - check that remaining buffer size is more or equal to `8`
   bytes. If shorter - RETURN invalid status `PARENT_ID_TOO_SHORT`.
   Otherwise read the next `8` bytes for `parent-id` and shift cursor
   to the end of those `8` bytes.
6. **Parse `trace-id`**. Read the field identifier byte at the cursor
   position. If NOT `2` - go to step `8. Report invalid field`.
   Otherwise - check the remaining size of the buffer. If at the end of
   the buffer - RETURN invalid status. Otherwise - read the
   `trace-flags` byte. Least significant bit will represent `recorded`
   value.
7. RETURN status `OK` if `version` is `0` or status `DOWNGRADED_TO_ZERO`
   otherwise.
8. **Report invalid field**.  If `version` is `0` RETURN invalid status
   `INVALID_FIELD_ID`. If `version` has any other value -
   `INCOMPATIBLE_VERSION`

_Note_, that invalid status names are given for readability and not part of the
specification.

_Note_, that parsing should not treat any additional bytes in the end of the
buffer as an invalid status. Those fields can be added for padding purposes.
Optionally implementation can check that the buffer is longer than `29` bytes as
a very first step if this check is not expensive.

## De-serialization of `tracestate`

Let's assume the algorithm takes a buffer - bytes array - and can set
and shift cursor in the buffer as well as validate whether the end of
the buffer was reached or will be reached after reading the given number
of bytes. Algorithm also uses `version` value parsed from `traceparent`.
If `version` was not given - value `0` SHOULD be used. This algorithm
can work on stream of bytes. De-serialization of `tracestate` MAY be
done in the following sequence:

1. If at the end of the buffer - RETURN status `OK`. Otherwise set a
   cursor to the first byte.
2. **Parse `list-member` field identifier**. Read the field identifier
   byte at the cursor position and shift cursor to `1` byte. If NOT `0`
   and `version` is `0` RETURN invalid status `INVALID_FIELD_ID`. If NOT
   `0` and `version` has any other value - `INCOMPATIBLE_VERSION`.
3. **Parse key**.
   1. If at the end of the buffer - RETURN status `OK`. This situation
      indicates that `tracestate` value was padded with `0`.
   2. Read the `key-len` byte. Shift cursor to `1` byte. If the value of
      `key-len` is `0` - RETURN status `OK`. This situation indicates an
      explicit end of a key.
   3. Check that buffer has `key-len` more bytes. If not - RETURN
      `KEY_TOO_SHORT`.
   4. Read `key-len` bytes as `key`. Shift cursor to `key-len` bytes.
4. **Parse value**.
   1. If at the end of the buffer - RETURN status `INCOMPLETE_LIST_MEMBER`.
   2. Read the `value-len` byte. Shift cursor to `1` byte. If the value of
      `value-len` is `0` - add `list-member` with the `key` and empty
      `value` to the `tracestate` list. RETURN status `OK`.
   3. Check that buffer has `value-len` more bytes. If not - RETURN
      `VALUE_TOO_SHORT`.
   4. Read `value-len` bytes as `value`. Shift cursor to `value-len`
      bytes.
   5. Add `list-member` with the `key` and `value` to the `tracestate`
      list.
5. Go to step `2. Parse list-member field identifier`.
