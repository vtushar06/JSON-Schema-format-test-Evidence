# JSON Schema `format` cross-implementation evidence

Inputs where real-world validators across languages disagree on a `format`, each grounded in the RFC the JSON Schema specification points at. Built to inform test-suite coverage.

Each finding gives the input, the expected verdict per the governing RFC, and which implementations accept it when they should not. Inputs that every validator already handles the same way are left out; the value is in the disagreements.

## Method

1. Read the governing RFC(s) for the format and list every grammar rule and boundary.
2. Run a wide input set through validators in several languages: Python (`ipaddress`, `jsonschema`), C libc (`inet_aton`, `inet_pton`), JavaScript (ajv-formats), Ruby (`IPAddr`), PHP (`filter_var`), Go (`net/netip`), Rust (`std::net`). Java Guava, .NET, and the WHATWG URL parser are checked separately and noted where they matter.
3. Record every input where implementations diverge, or where one diverges from the RFC.
4. Cross-check against published CVEs and parser bug reports.

## Reproduction

Every row can be re-checked with a one-line command. The exact snippets per implementation are in [`ipv4.md`](./ipv4.md#how-to-reproduce-any-row). Implementation versions used are listed there too.

## Scope

Verdicts follow the RFC the JSON Schema spec references for each format (for `ipv4`, RFC 2673 section 3.2). Where two RFCs conflict or an RFC is ambiguous (for example, leading zeros in `ipv4`), both readings are shown and the case is left without a verdict rather than forced into one.

## Files

- [`ipv4.md`](./ipv4.md) IPv4 dotted-quad findings (inet_aton shorthand, NUL injection, non-ASCII digits, leading-zero ambiguity).
- [`uuid.md`](./uuid.md) UUID findings (URN prefix accepted by ajv-formats, trailing-hyphen and non-ASCII digit accepted by python-jsonschema).
- More formats added as the work proceeds.
