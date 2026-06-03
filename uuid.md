# UUID cross-implementation findings

Governing grammar: **RFC 4122 section 3** (also RFC 9562 section 4), the RFC the JSON Schema specification references for `format: uuid` (spec section 7.3.5). The text representation is `8-4-4-4-12` ASCII hex digits separated by single hyphens. `HEXDIG` is `DIGIT / "A" / "B" / "C" / "D" / "E" / "F"` - ASCII-only. RFC 5234 section 2.3 makes the alphabetic characters case-insensitive, so `a-f` and `A-F` are both valid. The grammar does not restrict the version or variant nibble values; those are semantic, not syntactic. The Nil UUID and Max UUID are valid string representations.

The spec adds an explicit note (section 7.3.5): "the 'uuid' format is for plain UUIDs, not UUIDs in URNs. For UUIDs as URNs, use the 'uri' format."

---

## URN prefix accepted by ajv-formats

ajv-formats' regex (file `dist/formats.js`, version 3.0.1) is:

```
/^(?:urn:uuid:)?[0-9a-f]{8}-(?:[0-9a-f]{4}-){3}[0-9a-f]{12}$/i
```

The `(?:urn:uuid:)?` group makes the prefix optional. Combined with the `/i` flag, it also accepts `URN:UUID:` and `urn:uuid:` (any case). The spec is explicit that the `uuid` format does not cover URN-wrapped UUIDs.

python-jsonschema 4.25.1 correctly rejects all three variants because its position check `all(instance[p] == "-" for p in (8, 13, 18, 23))` operates on the original string - when the prefix is present, position 8 is `:` not `-`.

| input | oracle | python-jsonschema 4.25.1 | ajv-formats 3.0.1 |
|---|---|---|---|
| `urn:uuid:2eb8aa08-aa98-11ea-b4aa-73b441d16380` | invalid (plain UUID only) | reject | **accept** |
| `URN:UUID:2EB8AA08-AA98-11EA-B4AA-73B441D16380` | invalid | reject | **accept** |
| `2eb8aa08-aa98-11ea-b4aa-73b441d16380` | valid | accept | accept |

Reproduce:

```bash
# ajv-formats accepts the urn: prefix
node -e "
const Ajv=require('ajv'), af=require('ajv-formats');
const a=new Ajv({strict:false}); af(a);
const v=a.compile({type:'string',format:'uuid'});
console.log(v('urn:uuid:2eb8aa08-aa98-11ea-b4aa-73b441d16380'));  // true - BUG
console.log(v('2eb8aa08-aa98-11ea-b4aa-73b441d16380'));           // true - ok
"

# python-jsonschema correctly rejects
python3 -c "
from jsonschema import Draft202012Validator as V
c=V.FORMAT_CHECKER
print(c.conforms('urn:uuid:2eb8aa08-aa98-11ea-b4aa-73b441d16380','uuid'))  # False
print(c.conforms('2eb8aa08-aa98-11ea-b4aa-73b441d16380','uuid'))           # True
"
```

---

## Trailing hyphen accepted by python-jsonschema

A complete 36-character UUID with one extra hyphen at the end (37 characters total) is structurally invalid - the RFC grammar produces exactly 32 hex characters and 4 hyphens at fixed positions. python-jsonschema 4.25.1 accepts it; ajv-formats rejects it.

Mechanism: `is_uuid` calls `uuid.UUID(s)` then checks only that positions 8, 13, 18, and 23 of the original string are `-`. `uuid.UUID()` strips all hyphens before checking that 32 hex characters remain - it never validates hyphen count or that no extra characters exist. A trailing hyphen does not shift the checked positions (8/13/18/23 are all still `-`), so the position check passes. ajv-formats' `$` anchor rejects the extra character.

| input | oracle | python-jsonschema 4.25.1 | ajv-formats 3.0.1 |
|---|---|---|---|
| `2eb8aa08-aa98-11ea-b4aa-73b441d16380-` (37 chars) | invalid | **accept** | reject |

This is distinct from the existing "too many dashes" test in the suite (`2eb8-aa08-aa98-11ea-b4aa73b44-1d16380`), which python-jsonschema correctly rejects because the interior hyphens shift positions 13, 18, and 23.

Reproduce:

```bash
# python-jsonschema accepts the trailing hyphen
python3 -c "
from jsonschema import Draft202012Validator as V
c=V.FORMAT_CHECKER
print(c.conforms('2eb8aa08-aa98-11ea-b4aa-73b441d16380-','uuid'))  # True - BUG
print(c.conforms('2eb8aa08-aa98-11ea-b4aa-73b441d16380','uuid'))   # True - ok
"

# ajv-formats correctly rejects
node -e "
const Ajv=require('ajv'),af=require('ajv-formats');
const a=new Ajv({strict:false}); af(a);
const v=a.compile({type:'string',format:'uuid'});
console.log(v('2eb8aa08-aa98-11ea-b4aa-73b441d16380-'));  // false
"
```

---

## Non-ASCII digit accepted by python-jsonschema

A UUID where one hex-position digit is replaced by a visually similar non-ASCII Unicode decimal digit is structurally invalid per RFC 4122 - `HEXDIG` is `%x30-39 / "A"-"F"` (ASCII only). python-jsonschema 4.25.1 accepts it; ajv-formats rejects it.

Mechanism: python-jsonschema calls `uuid.UUID(s)`. Internally, `uuid.UUID.__init__` strips hyphens and calls `int(hex_str, 16)`. Python's built-in `int(..., 16)` delegates to the C `strtol` family and also accepts Unicode decimal digits that Python's `str.isdigit()` considers numeric - including Bengali, Devanagari, Arabic-Indic, fullwidth, and similar scripts. So `int("২", 16)` returns 2, the UUID parses, and the position check passes. ajv-formats' regex character class `[0-9a-f]` is ASCII-only and rejects immediately.

| input | oracle | python-jsonschema 4.25.1 | ajv-formats 3.0.1 |
|---|---|---|---|
| `২eb8aa08-aa98-11ea-b4aa-73b441d16380` (Bengali 2, U+09E8, at position 0) | invalid | **accept** | reject |

The same bug applies to fullwidth digits (U+FF10-FF19), Devanagari (U+0966-U+096F), Arabic-Indic (U+0660-U+0669), Thai (U+0E50-U+0E59), and others that Python `int()` maps to ASCII values.

Reproduce:

```bash
# confirm Python int() folds Bengali 2 to ASCII 2
python3 -c "print(int('২', 16))"  # 2

# python-jsonschema accepts the Bengali digit
python3 -c "
from jsonschema import Draft202012Validator as V
c=V.FORMAT_CHECKER
print(c.conforms('২eb8aa08-aa98-11ea-b4aa-73b441d16380','uuid'))  # True - BUG
print(c.conforms('2eb8aa08-aa98-11ea-b4aa-73b441d16380','uuid'))  # True - ok
"

# ajv-formats correctly rejects
node -e "
const Ajv=require('ajv'),af=require('ajv-formats');
const a=new Ajv({strict:false}); af(a);
const v=a.compile({type:'string',format:'uuid'});
console.log(v('২eb8aa08-aa98-11ea-b4aa-73b441d16380'));  // false
"
```

---

## Versions

python-jsonschema 4.25.1 (CPython 3.9.6, macOS); ajv 8.20.0 + ajv-formats 3.0.1 (Node 20.19.5, macOS). All runs on macOS (Darwin arm64).
