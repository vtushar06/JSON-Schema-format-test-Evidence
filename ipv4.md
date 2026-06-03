# IPv4 dotted-quad cross-implementation findings

Governing grammar: **RFC 2673 section 3.2** (`dotted-quad`, `decbyte = 1*3DIGIT`, value 0 to 255), the RFC the JSON Schema specification references for `format: ipv4`. Digits are ASCII 0 to 9 only, per RFC 5234 section B.1. Under this grammar every input below is invalid, yet at least one widely used validator accepts it.

Implementations exercised (versions at the bottom): python `ipaddress`, python `jsonschema`, C `inet_aton`, C `inet_pton`, ajv-formats, Ruby `IPAddr`, PHP `filter_var`, Go `net/netip`, Rust `std::net`. Java Guava, .NET, and the WHATWG URL parser were checked separately and are noted at the end.

Note: ajv-formats, the most widely used JSON Schema validator, rejects every input below. So these are not ajv bugs; they are tripwires for any implementation that delegates IPv4 checking to a lenient system parser.

Two platform notes:
- `inet_pton` results are from macOS (Darwin) libc, which accepts leading zeros. glibc `inet_pton` rejects them (RFC 6943 section 3.1.1). So "C `inet_pton` (Darwin)" below is a platform-specific result, not universal.
- `inet_aton` is lenient by design on every platform (RFC 6943 section 3.1.1 calls this the "loose" form).

---

## inet_aton short-form folding

`inet_aton` accepts fewer than four parts and folds the last part into the remaining bytes, producing a surprising address.

| input | accepted by | resolves to |
|---|---|---|
| `1.2` | C `inet_aton` | 1.0.0.2 |
| `1.16777215` | C `inet_aton` | 1.255.255.255 |
| `16909060` | C `inet_aton` | 1.2.3.4 |
| `0xc0a80101` | C `inet_aton` | 192.168.1.1 |

## Shorthand (fewer than four parts)

| input | accepted by | resolves to |
|---|---|---|
| `127` | C `inet_aton` | 0.0.0.127 |
| `127.1` | C `inet_aton` | 127.0.0.1 |
| `10.1` | C `inet_aton` | 10.0.0.1 |
| `192.168.257` | C `inet_aton` | 192.168.1.1 |
| `1.2.3` | C `inet_aton` | 1.2.0.3 |
| `10.0.258` | C `inet_aton` | 10.0.1.2 |

## 32-bit integer forms

| input | accepted by | resolves to |
|---|---|---|
| `2130706433` | C `inet_aton` | 127.0.0.1 |
| `3232235521` | C `inet_aton` | 192.168.0.1 |
| `0x7f000001` | C `inet_aton` | 127.0.0.1 |
| `0xffffffff` | C `inet_aton` | 255.255.255.255 |
| `0xA000102` | C `inet_aton` | 10.0.1.2 |

## Hexadecimal / alternate-base octets

| input | accepted by | resolves to |
|---|---|---|
| `0x7f.0.0.1` | C `inet_aton` | 127.0.0.1 |
| `0X7F.0.0.1` | C `inet_aton` | 127.0.0.1 |
| `0xc0.0xa8.0x01.0x01` | C `inet_aton` | 192.168.1.1 |
| `0x0a.0.0.1` | C `inet_aton` | 10.0.0.1 |

## Mixed radix per octet

| input | accepted by | resolves to |
|---|---|---|
| `0x8.0X8.010.8` | C `inet_aton` | 8.8.8.8 |
| `0251.00376.000251.0000376` | C `inet_aton` | 169.254.169.254 |

## Embedded NUL byte

A C string stops at the first NUL, so a lenient C parser sees only the part before it.

| input | accepted by | resolves to |
|---|---|---|
| `192.168.0.1\x00` | C `inet_aton`, C `inet_pton` (Darwin) | 192.168.0.1 |
| `192.168.0.1\x00.evil.com` | C `inet_aton`, C `inet_pton` (Darwin) | 192.168.0.1 |

## Trailing characters after a valid address

| input | accepted by | resolves to |
|---|---|---|
| `127.0.0.1\r\nspam` | C `inet_aton` | 127.0.0.1 |
| `1.1.1.1 wtf` | C `inet_aton` | 1.1.1.1 |
| `0.0.0.0xff` | C `inet_aton` | 0.0.0.255 |
| `192 168 0 1` | C `inet_aton` | 0.0.0.192 |
| `255.255.255.255 ` | C `inet_aton` | 255.255.255.255 |

## Leading-zero octets (octal ambiguity, platform-specific)

These are the inputs left without a final verdict in the grammar (RFC 2673 allows the digits, but RFC 3986 and most parsers forbid them). Listed here because the two C functions on Darwin accept them while the others reject them.

| input | accepted by |
|---|---|
| `01.2.3.4` | C `inet_aton`, C `inet_pton` (Darwin) |
| `010.0.0.1` | C `inet_aton`, C `inet_pton` (Darwin) |
| `0177.0.0.1` | C `inet_aton`, C `inet_pton` (Darwin) |
| `00.0.0.0` | C `inet_aton`, C `inet_pton` (Darwin) |
| `192.168.000.001` | C `inet_aton`, C `inet_pton` (Darwin) |

## Ruby IPAddr accepts CIDR suffixes

| input | accepted by |
|---|---|
| `192.168.1.0/24` | Ruby `IPAddr` |
| `192.168.1.0/0` | Ruby `IPAddr` |
| `192.168.0.1/` | Ruby `IPAddr` |

## Non-ASCII digits (Java Guava and the WHATWG URL parser)

The nine local validators reject every non-ASCII digit. Two parsers checked separately do not:

- `Guava InetAddresses.forString` accepts seven non-ASCII digit scripts (fullwidth, Arabic-Indic, extended Arabic-Indic, Devanagari, Bengali, Thai, Tamil), each resolving to the matching ASCII address, because it reads digits with a Unicode-aware routine (`Character.digit`).
- The WHATWG URL parser accepts a different set (fullwidth, mathematical-bold, monospace, superscript, circled) because it applies Unicode compatibility mapping before parsing.

No single non-ASCII script is accepted by both. For example `१९२.168.0.1` (Devanagari) is accepted by Guava but rejected by the WHATWG parser and by every strict validator. This is a homograph surface.

### Two inputs that expose the real-world split

These are the exact strings that motivated the JSON Schema Test Suite addition. All verified live against installed implementations.

| input | WHATWG URL (Node 20) | Java Guava 33.3.1 | ajv-formats 3.0.1 |
|---|---|---|---|
| `１９２.１６８.１.１` (fullwidth digits) | accept - resolves to 192.168.1.1 | accept - resolves to 192.168.1.1 | reject |
| `𝟏𝟗𝟐.𝟏𝟔𝟖.𝟏.𝟏` (mathematical-bold, astral-plane) | accept - resolves to 192.168.1.1 | reject | reject |

The fullwidth case is accepted by both a Unicode-aware Java library (Guava, via `Character.digit`) and a browser-spec URL parser (Node's `new URL()`). The mathematical-bold case (astral-plane, surrogate pair in UTF-16) slips past Guava's `charAt` check but is still folded by the WHATWG UTS-46 mapping. ajv-formats rejects both because its regex `[0-9.]` is ASCII-only.

The risk: an application that format-validates with ajv-formats and then resolves the host with `new URL(input).hostname` for an allowlist check sees two different strings.

Reproduce the WHATWG case (Node 20):

```bash
node -e "console.log(new URL('http://１９２.１６８.１.１/').hostname)"
# 192.168.1.1
node -e "console.log(new URL('http://𝟏𝟗𝟐.𝟏𝟔𝟖.𝟏.𝟏/').hostname)"
# 192.168.1.1
```

Reproduce the Guava case (Java 21 `jshell`, with a Guava jar on the class path - point `--class-path` at your local `guava-*.jar`):

```bash
# fullwidth - accepted, returns /192.168.1.1
echo 'System.out.println(com.google.common.net.InetAddresses.forString("１９２.１６８.１.１"))' \
  | jshell --class-path guava-33.3.1-jre.jar -q

# mathematical-bold - rejected, throws IllegalArgumentException
echo 'System.out.println(com.google.common.net.InetAddresses.forString("𝟏𝟗𝟐.𝟏𝟔𝟖.𝟏.𝟏"))' \
  | jshell --class-path guava-33.3.1-jre.jar -q
```

Reproduce ajv-formats rejection:

```bash
node -e "const f=require('ajv-formats/dist/formats').fullFormats;
console.log(f.ipv4.test('１９２.１６８.１.１'));  // false
console.log(f.ipv4.test('𝟏𝟗𝟐.𝟏𝟔𝟖.𝟏.𝟏'));  // false"
```

## Same string, two different addresses

`0177.0.0.1` is read as `127.0.0.1` by parsers that treat a leading `0` as octal (inet_aton, browsers), but as `177.0.0.1` by parsers that ignore the leading zero. A filter that classifies `177.x.x.x` as public while the OS connects to `127.0.0.1` is the basis of CVE-2021-28918.

---

## How to reproduce any row

Replace `INPUT` with the row's input. A non-zero / true / no-exception result means the validator accepted it.

`inet_aton` and `inet_pton` (shows accept plus the resolved address):

```bash
python3 - <<'PY'
import ctypes, ctypes.util, socket
libc = ctypes.CDLL(ctypes.util.find_library("c"))
class S(ctypes.Structure): _fields_=[("a", ctypes.c_uint32)]
s = S(); buf=(ctypes.c_byte*4)()
inp = b"INPUT"
aton = libc.inet_aton(inp, ctypes.byref(s))
print("inet_aton:", "accept ->", socket.inet_ntoa(s.a.to_bytes(4,"little")) if aton else "reject")
print("inet_pton:", "accept" if libc.inet_pton(2, inp, buf)==1 else "reject")
PY
```

python `ipaddress` and `jsonschema`:

```bash
python3 -c "import ipaddress; print(ipaddress.IPv4Address('INPUT'))"
python3 -c "from jsonschema import Draft202012Validator as V; V.FORMAT_CHECKER.check('INPUT','ipv4'); print('accept')"
```

ajv-formats:

```bash
node -e "const f=require('ajv-formats/dist/formats').fullFormats; console.log(f.ipv4.test('INPUT'))"
```

Ruby `IPAddr`, PHP `filter_var`:

```bash
ruby -ripaddr -e 'begin; puts IPAddr.new(ARGV[0]).ipv4?; rescue; puts false; end' 'INPUT'
php -r 'var_dump(filter_var("INPUT", FILTER_VALIDATE_IP, FILTER_FLAG_IPV4) !== false);'
```

Go `net/netip`, Rust `std::net`:

```bash
go run - <<'GO'
package main
import ("fmt";"net/netip")
func main(){ a,e:=netip.ParseAddr("INPUT"); fmt.Println(e==nil && a.Is4()) }
GO
echo 'fn main(){ println!("{}", "INPUT".parse::<std::net::Ipv4Addr>().is_ok()); }' > /tmp/r.rs && rustc -o /tmp/r /tmp/r.rs && /tmp/r
```

## Versions

macOS (Darwin) libc; Node 18 (ajv-formats latest); Python 3.9 (`ipaddress` stdlib, `jsonschema` 4.25); Ruby 2.6 `IPAddr`; PHP 8.1; Go 1.x `net/netip`; Rust 1.94 `std::net`; Java 21 (Guava 33.x); whatwg-url 14.x. inet_pton acceptance of leading zeros is specific to Darwin libc; glibc rejects.
