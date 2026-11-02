# hlib

The `.hlib` archive format (spec: [docs/HLIB_FORMAT.md](docs/HLIB_FORMAT.md)) implemented **once**, in H#,
as a [bit.io](https://github.com/HackerOS-Linux-System) package. It replaces the three private copies
in H#, Hacker Lang and HackerScript.

- **Library** (`src/lib.h#`): open / check / extract / build / sign / search.
- **CLI** (`hlib`): `build inspect list verify extract manifest cat keygen find import`.
- **HackerScript binding** (`lib/mod.hcs`): same five functions as the native block in `hackerc/cmd/hlib.hcs`.

## Use from H#

```
# Bit.hk
[dependencies]
-> hlib => path ../hlib        # or the index entry, pinned to a rev
```

```
use "hlib" from "hlib"

fn main() -> int is
    let r = hlib::open("foo.hlib")
    let a = r?
    let rep = hlib::check_strict(a, "")?     ;; "" = no trusted key
    if !rep.ok is return 1 end
    return 0
end
```
A signature counts as *trusted* only when verified against a key you pass; see [SECURITY.md](docs/SECURITY.md).

## CLI

```
hlib keygen --out my.key
hlib build -o foo.hlib --name foo --version 1.0.0 --lang hsharp --so x86_64-unknown-linux-gnu=libfoo.so --sign my.key
hlib verify foo.hlib --pubkey <hex>        # strict by default; --compat for the Rust-reader rules
hlib extract foo.hlib --into out/          # verifies first
```
Exit codes: 0 ok, 1 operation failed, 2 usage error.

## Build and test

Needs libzstd and libsodium (static, headers + libs).
```
tools/test.sh          # unit tests (21) + interop (python3-zstandard, pynacl; HLIB_RUST_REF optional)
bit build              # produces the hlib binary
```

## Docs
[HLIB_FORMAT](docs/HLIB_FORMAT.md) · [API](docs/API.md) · [BOOTSTRAP](docs/BOOTSTRAP.md) · [SECURITY](docs/SECURITY.md)

## Known limits
- H# 0.9 cannot export a C ABI (spec §11), so HackerScript uses the CLI, not FFI.
- `lib/mod.hcs` and the `[system-dependencies]` syntax in `Bit.hk` were not run against a real HackerScript compiler / bit.
