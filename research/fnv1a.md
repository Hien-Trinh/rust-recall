# FNV-1a 64-bit: constants and test vectors

Answers issue #3. These are the facts the M2 hash tests are built on. No implementation here.

## Sources

- **[RFC 9923]** L. Noll, K. Vo, D. Eastlake 3rd, T. Hansen, *The FNV Non-Cryptographic Hash Algorithm*, RFC 9923 (Informational, Independent Submission), February 2026. https://www.rfc-editor.org/rfc/rfc9923.html. It is the published form of `draft-eastlake-fnv`.
- **[isthe]** Landon Curt Noll's FNV page: http://www.isthe.com/chongo/tech/comp/fnv/index.html. It points to the reference code at https://github.com/lcn2/fnv (`test_fnv.c`).

## Constants (64-bit)

| Constant | Decimal | Hex | Source |
|---|---|---|---|
| `FNV_prime` | 1099511628211 | `0x00000100000001B3` | RFC 9923 §5, isthe |
| `offset_basis` | 14695981039346656037 | `0xCBF29CE484222325` | RFC 9923 §5, isthe |

The prime is also written as 2^40 + 2^8 + 0xB3 (RFC 9923 §5). I checked that `2**40 + 2**8 + 0xb3 == 0x100000001b3 == 1099511628211`.

## Algorithm (FNV-1a)

From RFC 9923 §2 (isthe gives the same steps):

```
hash = offset_basis
for each octet_of_data to be hashed
    hash = hash XOR octet_of_data
    hash = hash * FNV_Prime mod 2**HashSize
return hash
```

- The loop works on **octets (bytes)**, not characters. For a Rust `&str`, that means its UTF-8 bytes.
- The XOR comes **before** the multiply. FNV-1 does them in the other order, and mixing the two up is the classic bug.
- `mod 2**64` means the multiply wraps around in u64.
- The hash is the u64 value itself. RFC 9923 §2.3 covers byte order, which only matters if you serialize the hash as bytes.

## Test vectors

The RFC rows come from RFC 9923 §8.3, `Test64()`, array `FNV64svalues`. The test strings are `teststring[] = { "", "a", "foobar", "Hello!\x01\xFF\xED" }`, and these are string hashes with no terminating NUL.

| Input | Bytes (hex) | FNV-1a 64 | Source | Recomputed |
|---|---|---|---|---|
| `""` | (none) | `0xcbf29ce484222325` | RFC 9923 §8.3 | match |
| `"a"` | `61` | `0xaf63dc4c8601ec8c` | RFC 9923 §8.3 | match |
| `"foobar"` | `66 6f 6f 62 61 72` | `0x85944171f73967e8` | RFC 9923 §8.3 | match |
| `"Hello!\x01\xFF\xED"` | `48 65 6c 6c 6f 21 01 ff ed` | `0xbd51ea7094ee6fa1` | RFC 9923 §8.3 | match |
| `"café"` (UTF-8) | `63 61 66 c3 a9` | `0x48e8823acfa40d89` | computed | n/a |
| `"日本語"` (UTF-8) | `e6 97 a5 e6 9c ac e8 aa 9e` | `0xee9ee2b5c854ef87` | computed | n/a |
| `"x²"` (UTF-8) | `78 c2 b2` | `0xc0e4fc1980f24f17` | computed | n/a |

Notes:

- **The empty string hashes to `offset_basis`.** That makes it a good first test.
- **The RFC's non-ASCII vector is not valid UTF-8.** `0xFF` never appears in UTF-8, so a Rust `&str` cannot hold that input. Test it through a `&[u8]` entry point if the hash takes bytes. Otherwise, skip it.
- **No primary source publishes a UTF-8 vector.** I computed the three UTF-8 rows with the RFC algorithm (a throwaway `python3 -I` script). The same script reproduced all four RFC vectors exactly, so the UTF-8 rows can be trusted as far as the algorithm can.
- **Watch out for the RFC's second array.** `FNV64bvalues` in the RFC hashes each string *with* its trailing NUL (`strlen+1`). For example, `"foobar\0"` gives `0x34531ca7168b8f38`, which I also reproduced. Don't use those values for card hashing.
- isthe also lists inputs that hash to 0. One example is the bytes `d5 6b b9 53 42 87 08 36`, which give `FNV64_1a = 0`. These make good edge-case tests if wanted.

## Verification script used

```
python3 -I -c '
P=0x100000001b3; B=0xcbf29ce484222325
def h(b):
  x=B
  for o in b: x=((x^o)*P)%2**64
  return x
for s in [b"",b"a",b"foobar",b"Hello!\x01\xff\xed","café".encode(),"日本語".encode(),"x²".encode()]:
  print(repr(s), "%016x"%h(s))'
```

There were no mismatches between this script and the RFC's published values.
