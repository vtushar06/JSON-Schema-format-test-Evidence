# idn-hostname format - cross-implementation evidence

`format: idn-hostname` is governed by RFC 5890/5891/5892/5893 (IDNA2008) + RFC 1034. The notes
below are inputs the existing JSON Schema test suite does not cover, each invalid under IDNA2008
yet accepted by at least one real implementation. Every verdict is the strict IDNA2008 reading;
every "accepts/rejects" is from an actual run.

Implementations tested (9): python `idna` 3.10 (used by python-jsonschema), GNU libidn2 2.3.8,
Go `golang.org/x/net/idna` v0.55, Node 20 WHATWG `url.domainToASCII`, PHP 8.1 `idn_to_ascii`
(ICU UTS46), Rust `idna` 1.1 crate, Java 21 `java.net.IDN`, ICU4J 76.1 (strict flags), Guava 33.4.
Spec lineages: IDNA2008-strict (python), UTS46 (libidn2/Go/Node/PHP/Rust/ICU4J), IDNA2003 (Java).

## A-label that decodes to a DISALLOWED code point

`xn--7a` is well-formed Punycode that decodes to U+00A1, which is DISALLOWED under IDNA2008
(RFC 5892 §2.6). An A-label is only valid if it decodes to a valid U-label (RFC 5890 §2.3.2.1).
Accepted by Go, Node, PHP, Rust, Java, ICU4J and Guava (7); python idna rejects.
Reproduce: `node -e "console.log(require('url').domainToASCII('xn--7a'))"` prints `xn--7a`.

## A-label that decodes to a pure-ASCII / empty label

`xn--example-` decodes to `example` (all ASCII); `xn--` decodes to empty. A U-label must contain
at least one non-ASCII character (RFC 5890 §2.3.2.1), so an A-label that decodes to all-ASCII is
invalid (codified by UTS46 revision 33). This class is CVE-2024-12224 (Rust idna), CVE-2026-39821
(Go x/net/idna) and CVE-2026-46644 (PHP symfony polyfill). Go x/net and Node currently accept it.

## Non-canonical Punycode (round-trip failure)

`xn---9uc` decodes to a single Kannada code point whose canonical A-label is `xn--9uc` (no third
hyphen). RFC 5891 §5.4 requires a decoded label to re-encode to itself. python idna and Node accept
the non-canonical form; Go, PHP, Rust and ICU4J reject.

## Fullwidth digits, zero-width and other DISALLOWED code points

`１２３` (U+FF11..U+FF13, fullwidth digits) and `a`+U+200B+`b` (zero width space) are DISALLOWED
under IDNA2008 (RFC 5892 §2.6). UTS46 processors map fullwidth digits to ASCII and treat U+200B as
ignorable (stripping it). Both accepted by libidn2, Go, Node, PHP, Rust, Java, ICU4J and Guava (8);
python idna rejects. U+200B silent-strip is also GNU libidn2 GitLab issue #136.

## Whole-name Bidi rule

`0a.א` is a Bidi domain name (it has the RTL label `א`), so every label must satisfy the RFC 5893
§2 Bidi rule - and `0a` starts with a digit, violating condition 1 (IdnaTestV2 marks it `[B1]`).
python idna accepts it: it checks the Bidi rule per-label and the `0a` label, seen alone, has no
RTL character, so the check is skipped. Node accepts it too. The reference C implementation
(sourcemeta/core) and Go reject it correctly.

## A-label longer than 63 octets

The 63-octet label limit is on the A-label form, not the code point count (RFC 5891 §4.2.4). 60 `ü`
is 60 code points but its A-label exceeds 63 octets. Go x/net and Node accept; python idna rejects.

## CONTEXTJ evaluated per occurrence

RFC 5892 appendix A.1 (erratum 3312): the ZWNJ rule must hold at every occurrence in a label. A
label with one ZWNJ in a valid post-Virama context and one between two ASCII letters is invalid;
Node and PHP accept it even though Node rejects the single-violation case.

## Reserved LDH labels

`ab--cd` is a pure-ASCII label with `--` in positions 3-4 and a prefix other than `xn--`. RFC 5890
§2.3.2.2 defines such a `??--` form as an R-LDH (reserved) label, and §2.3.2.3 limits an IDN to
NR-LDH / A-labels / U-labels. python idna rejects it; several UTS46 processors and sourcemeta/core
accept it (it is also a valid plain RFC 1123 hostname, so this one is a genuine spec-interpretation
question rather than a clear error).
