# Text Decode And Mod Notes

This note records the current understanding of the text decoding path used by
the local Uncharted Waters II research scripts, especially the relationship
between the offset table, the `0x00` terminator, and safe text modding space.

## Scope

The concrete implementation discussed here is:

- `sea2_demo/scripts/decode_text.py`
- source files such as `Message.dat` and `Snr*.mes`
- custom character mapping file `KoeiCht.txt`

This is about text decoding, not LS11/LZW resource decompression.

## Text File Structure

The text files begin with an offset table. The offset width depends on the file:

- `Message.dat`: big-endian `u16` offsets
- `Snr*.mes`: big-endian `u32` offsets

The first offset points to the beginning of the text body. Because the offset
table starts at byte `0`, the number of entries can be inferred from:

```text
entry_count = first_offset / offset_width
```

The decoder reads all offsets, appends `len(file)` as a synthetic final offset,
then slices each record as:

```text
record_i = file[offset[i] : offset[i + 1]]
```

So the offset table gives the storage boundary for each text record.

## Character Decoding

Inside each record, the decoder scans from left to right.

First it tries to read two bytes as a game character code:

```text
code = byte[i] << 8 | byte[i + 1]
```

That code is looked up in `KoeiCht.txt`. The mapping file stores:

```text
raw GBK bytes <TAB> game two-byte code
```

If the two-byte code exists in the mapping table, the raw bytes are decoded as
GBK and appended as one visible character.

If the two-byte lookup fails, the decoder falls back to one-byte handling:

- printable ASCII is emitted directly, including placeholders such as `$n`,
  `$s`, or `%s`
- `0x0a` is treated as a newline
- unknown bytes are preserved as `<xx>` markers

## Why `0x00` Still Matters

Even though the offset table already gives each record's byte range, `0x00` is
still meaningful. It is the in-string terminator.

The two boundaries answer different questions:

```text
offset[i]..offset[i + 1]  = where this stored record lives in the file
0x00                     = where the actual string content ends at runtime
```

This matches the common DOS-era string model: the game can pass a pointer to the
start of a string and the display routine reads until `0x00`.

Therefore, the current decoder stops on `0x00` to avoid interpreting terminator,
padding, or trailing bytes as text.

## Modding Implications

The space after `0x00` and before `offset[i + 1]` may be useful, but only within
limits.

Conceptually:

```text
[ offset[i] ................................ offset[i + 1] )
  visible string bytes + 00 + possible padding/free bytes
```

This trailing region is usually suitable for extending the current string in
place if all of the following are true:

- the new encoded string plus its final `0x00` stays within
  `offset[i + 1] - offset[i]`
- the bytes after the original `0x00` look like padding or unused space
- no script, table, or pointer targets an address inside the trailing region
- the offset table is left unchanged

Example safe shape:

```text
before: AA BB CC 00 00 00 00
after : AA BB CC DD EE 00 00
```

The new string consumes padding but does not cross the next record boundary.

This region is not automatically useful for independent new strings, because
the game usually only knows the original record start address. To make a new
string live after the old terminator, some other pointer, script operand, or
offset entry must be changed to point there.

For large text rewrites, the safer long-term strategy is to rebuild the text
body and regenerate the offset table, rather than relying on scattered padding
after `0x00`.

## Practical Rules

For conservative text mods:

1. Decode the target record through the existing offset table.
2. Measure the record capacity as `offset[i + 1] - offset[i]`.
3. Encode the replacement text with the same game character mapping.
4. Ensure `encoded_length + 1 <= record_capacity`.
5. Write one final `0x00` terminator.
6. Preserve or zero-fill the remaining bytes inside the same record.
7. Do not cross into the next record unless the offset table is rebuilt.

For aggressive mods:

1. Repack all changed text records into a new body.
2. Recalculate every offset.
3. Verify all script references still use text indices or updated pointers.
4. Test in-game text display, especially placeholders and line breaks.

