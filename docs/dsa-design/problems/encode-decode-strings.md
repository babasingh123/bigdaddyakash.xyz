# Design Encode and Decode Strings

!!! note "The disguise"
    Interviewer says: **"Design a codec that serializes a list of strings into a single string and deserializes it back"**
    They mean: **"Pick an unambiguous string-framing scheme — length-prefix is the canonical answer"**

## Problem

Implement:

- `encode(strs: list[str]) -> str`
- `decode(s: str) -> list[str]`

Strings may contain *any* character — including delimiters you might try to use.

## What it really tests

- Spotting the trap: a naïve delimiter like `","` or even `"#"` breaks the moment data contains that character.
- Designing a small framing protocol — exactly what HTTP, Protobuf, and Redis serializers do.
- Two options:
    1. **Length-prefix**: `"<len>#<payload>"`. Robust; no escaping needed.
    2. **Escape characters**: pick a delimiter, escape it inside payloads. Tedious and bug-prone.

## Approach

Use length-prefix encoding. For each string `s` write `f"{len(s)}#{s}"`. To decode, scan to the next `#`, parse the length, take that many characters, repeat.

Why is this correct even when the payload contains `#`? Because we don't search for `#` *globally* — we only look for the first `#` after the current cursor, which is always the length terminator. The payload is then consumed by *count*, not by scanning.

Why not JSON? It works but it is heavier and requires another library. The interviewer wants to see that you can design framing yourself.

## Code sketch

```python
class Codec:
    def encode(self, strs: list[str]) -> str:
        return "".join(f"{len(s)}#{s}" for s in strs)

    def decode(self, s: str) -> list[str]:
        out, i = [], 0
        while i < len(s):
            j = s.index("#", i)
            n = int(s[i:j])
            out.append(s[j + 1 : j + 1 + n])
            i = j + 1 + n
        return out
```

## Complexity

- `encode`: O(total characters).
- `decode`: O(total characters).
- Space: O(total characters).

## Edge cases

- Empty list → `""`. Decode of `""` → `[]`.
- Empty string in the list → encoded as `"0#"`.
- Strings containing `#`, newlines, or non-ASCII characters — all fine because we never search past the length prefix.
- Multi-byte: works in Python where `str` is character-indexed; in Java use byte lengths to be safe.

## Linked concepts

- [String & Pattern problems](../by-category/string-pattern.md)
- The same framing idea drives wire protocols — see [API Design fundamentals](../../hld/fundamentals/api-design.md).
