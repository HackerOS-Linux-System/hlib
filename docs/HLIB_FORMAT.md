# `.hlib` — HackerOS Lib

**Status:** v1 (`hlib_spec_version: 1`), text revision 2 (see §10)
**Scope:** a single, language-neutral library-archive format shared by three independent HackerOS language toolchains — **H#**, **Hacker Lang**, and **HackerScript**. Each toolchain ships its own independent, unpublished implementation of this spec (see "Why three implementations, not one crate" below) — this document is the one thing genuinely shared between them.

## 1. Container

A `.hlib` file is a `tar` archive, whole-stream compressed with `zstd` (level 19 by default). That's it — no custom magic bytes, no bespoke binary framing. `file mylib.hlib` correctly reports "Zstandard compressed data"; `zstd -d mylib.hlib -o mylib.tar && tar tf mylib.tar` works with stock tools for manual inspection.

## 2. Entries

| Path | Required | Purpose |
|---|---|---|
| `manifest.json` | yes | Everything below, in one place |
| `CHECKSUMS.sha256` | yes | `<hex-sha256>  <path>` per line, one per artifact, **sorted by path** |
| `SIGNATURE.ed25519` | no | Raw 64-byte Ed25519 signature over `CHECKSUMS.sha256`'s exact bytes |
| `native/<target-triple>/<name>.so` | no | Compiled shared object, one per target |
| `bytecode/<name>.hlbc` | no | Hacker Lang bytecode |
| `ast/...` | no | Producer-language source-level representation (see §5 — encoding differs per language) |
| `headers/<name>.iface.json` | no | Standalone copy of `manifest.exports`, for tooling that wants just the interface |
| `extra/...` | no | Anything else (docs, license, ...), passed through unmodified |

`manifest.json` is **not** included in `CHECKSUMS.sha256` — see §4 for why.

## 3. `manifest.json`

```jsonc
{
  "hlib_spec_version": 1,
  "name": "mylib",
  "version": "1.2.0",
  "language": "hsharp",          // "hsharp" | "hackerlang" | "hackerscript"
  "language_version": "0.9.0",   // producer toolchain version, informational
  "abi_version": 1,              // native-ABI generation of any `.so` artifacts
  "description": "...",
  "authors": ["..."],
  "created_at": "2026-09-16T12:00:00Z",
  "dependencies": [ { "name": "otherlib", "version_req": "^1.0" } ],
  "artifacts": [
    {
      "path": "native/x86_64-unknown-linux-gnu/mylib.so",
      "kind": "shared_object",        // shared_object | bytecode | ast | header | extra
      "target": "x86_64-unknown-linux-gnu",
      "sha256": "…",
      "size": 48213
    }
  ],
  "exports": [
    {
      "name": "add",
      "kind": "function",             // function | struct | enum | trait | const | type_alias
      "signature": "fn add(a: i64, b: i64) -> i64",
      "params": [ { "type": "i64" }, { "type": "i64" } ],
      "returns": { "type": "i64" },
      "generic": false,
      "is_macro": false
    }
  ],
  "signature": {                      // absent entirely if unsigned
    "algorithm": "ed25519",
    "public_key": "…",                // hex, 32 bytes
    "signed_entry": "CHECKSUMS.sha256"
  }
}
```

### The ABI-stable type set (`AbiType`)

`exports[].params`/`.returns` are lowered to a small, deliberately boring set every HackerOS language agrees maps to the same bit pattern: `i8 i16 i32 i64 u8 u16 u32 u64 f32 f64 bool void`, plus `ptr` (a raw byte pointer — strings/buffers, paired with a length per the producer's own convention) and `opaque { struct_name }` (an unresolved-layout handle, always passed by reference). Anything richer — generics, closures, language-specific struct layouts — has no entry here at all; it only exists in the `ast` artifact and is re-specialized by the *consumer's own* compiler.

## 4. Checksums and signing

`CHECKSUMS.sha256` covers every **artifact** entry (`native/…`, `ast/…`, `bytecode/…`, `headers/…`, `extra/…`) — deliberately **not** `manifest.json` itself. `manifest.json` already embeds each artifact's own `sha256`; excluding it sidesteps a circularity, since the final manifest bytes depend on whether a `signature` block is present, and that signature is computed *from* `CHECKSUMS.sha256`. A reader that wants to confirm `manifest.json` matches the archive it shipped with re-checks each `artifacts[].sha256` against `CHECKSUMS.sha256`'s lines — exactly what every reference reader's `verify_checksums()` does.

Signing is optional. When present, `SIGNATURE.ed25519` is a raw 64-byte Ed25519 signature over `CHECKSUMS.sha256`'s exact bytes. Verifying a signature only proves *this checksums file was signed by the holder of that private key* — it says nothing about whether that key should be trusted. Trust comes from the caller comparing the public key against one it already trusts (a lockfile pin, a declared publisher key, ...), the same way a package manager pins hashes today. No implementation invents or requires a trust store; that's left to each language's own package manager.

## 5. The `ast` artifact — one spec, three encodings

Every consumer needs a way to use a library's generics, macros, or (for source-hosted languages) its everyday functions without a rigid fixed-width ABI. That's what the `ast` artifact is for — but each language stores it in whatever shape its *own* front-end already parses, rather than forcing a shared cross-language AST schema that would need constant renegotiation as any one compiler evolves:

- **H#** (`ast/<name>.ast.json`): JSON array of `hsharp_parser::ast::Item` — the compiler's real AST type, which already derives `Serialize`/`Deserialize`. A consumer (H# itself) deserializes it back into real `Item`s and splices them into the program, byte-for-byte the same type the parser would have produced from source.
- **Hacker Lang** (`ast/<name>.ast.json`): JSON array of `hl_parser::Node` — same idea. Hacker Lang has no static type system, so this is usually the *primary* compiled artifact, not a fallback: its top-level `FuncDef`/`ArenaFuncDef` nodes are both the interface and something the runtime can execute directly.
- **HackerScript** (`ast/<name>.hcs`): the **original `.hcs` source text**, UTF-8, verbatim. HackerScript is self-hosted and still under active bootstrap, so its own AST node shapes change often; source text is the one representation its compiler is guaranteed to parse correctly at any point in its own development, with no separate schema to keep in sync. A consumer just parses this file the same way it parses any other `.hcs` file.

None of these are read cross-language. A Hacker Lang program consuming an H#-built `.hlib` never touches the `ast` artifact — it uses the `.so` + `headers` path instead (§6). The `ast` artifact is strictly a same-language fast path.

## 6. Consuming a `.hlib` — no `extern` required

Every language's compiler supports importing a `.hlib` directly (`use "hlib -> name"` in H#, `# <hlib/name>` in Hacker Lang, `get <hlib:name>` in HackerScript) without the program author writing any FFI declaration by hand. Resolution always tries two paths, in order:

1. **Same-language splice.** If the archive carries an `ast` artifact in *this* language's own encoding, the compiler deserializes/parses it and inlines the result directly into the program — same mechanism as a `mod`/`include` of a local file. Generics, macros, everything just works, because by the time typecheck runs, it *is* ordinary source in that language.
2. **Synthesized `extern` (fallback).** If there's no same-language `ast` artifact — a closed-source `.hlib`, or one produced by a *different* language — but there is a `.so` built for the host target, the compiler generates the FFI declaration **itself**, in memory, straight from `manifest.exports`, and links against the extracted `.so`. The program author still only wrote the one-line import.

Only non-generic function exports can be bound this way (`generic: true` exports need the `ast` path — there is no fixed-width call signature to generate for them). If neither path applies, the import fails with a clear error rather than silently doing nothing.

**Source-agnostic form.** A bare import (no explicit `hlib` source) resolves a `.hlib` first and falls back to an installed `bit` library of the same name:

- In HackerScript `get <mold>` (or `get <mold::1.2.0>`) needs no `source:` prefix. Right after a file is parsed, the project resolver rewrites it to `hlib` when a `.hlib` is found and otherwise to `bit` when a `bit` library of that name exists; if neither exists you get one warning naming both sources. `get <hlib:mold>` keeps working as the explicit form.
- In Hacker Lang `# <mold>` (or `# <mold:1.2.0>`) needs no namespace: the interpreter looks for a `.hlib` first (`hlibs/` of the project and its parents, the `bit` library directory, `~/.hackeros/Hacker-Lang/hlibs`, `/usr/lib/HackerOS/Hacker-Lang/hlibs`) and falls back to an installed `bit` library of the same name. `# <hlib/mold>` selects the `.hlib` explicitly. A same-language archive has its `ast` artifact deserialized and executed directly, so its functions are registered as if written in the program.

## 7. ABI stability — read this before assuming a `.so` is portable across languages

A compiled `.so`'s *symbol names* are consistent (plain, unmangled — H# and HackerScript both emit plain C-visible symbols for `pub`/exported functions), but each language's **calling convention for that symbol** is not automatically the same:

- **H#** and **HackerScript** exports use ordinary per-function typed C ABI — a `fn add(a: i64, b: i64) -> i64` is exactly that at the symbol level.
- **Hacker Lang**'s own FFI is a single fixed dispatch entrypoint, `hl_extern_call(argc: i32, argv: *const *const i8) -> i32` — string-marshalled, not a typed per-function ABI. A Hacker Lang program linking a foreign `.so` calls through that one entrypoint regardless of what the header lists; it does not call `add` directly by symbol.

Practically: H#-built and HackerScript-built `.so`s can usually be linked directly by each other's synthesized-`extern` path (§6.2). Calling either from Hacker Lang, or calling a Hacker Lang-oriented `.so` from H#/HackerScript, needs a small glue shim matching Hacker Lang's entrypoint convention — the format does not (yet) hide this difference. `manifest.exports[].signature` always tells you, in the producer's own words, what you're actually linking against.

## 8. Search paths

Every implementation searches, in order: a project-local `hlibs/` directory (walking up from the importing file), a package-manager-managed location when the ecosystem has one (H#'s `bytes` projects, HackerScript's `vira`-installed packages may ship a `.hlib` directly inside their installed directory), a per-user cache (`~/.hackeros/<Language>/hlibs/`), and a system-wide location (`/usr/lib/HackerOS/<Language>/hlibs/`). A `.hlib` sitting inside a `vira`/`bit`-installed package directory is picked up automatically — package-manager libraries are not a separate format from standalone `.hlib` files, just one more place the same file is looked for.

## 9. Implementations

The format is implemented by the `hlib` library (H#, bit.io — `bit install hlib`), which also ships the
`hlib` command line tool and a HackerScript binding. It supersedes the three independent copies
(`h#` `source-code/hlib`, Hacker Lang `hl-hlib`, HackerScript `hackerc/cmd/hlib.hcs`); until a
toolchain has switched, its copy must follow the clarifications below. The Rust reference
(`hl-hlib`) and `hlib` read each other's archives byte-compatibly (tests/interop.py).

## 10. Clarifications (normative from text revision 2; the format itself is unchanged, v1)

These pin down behaviour the original text left open, so that every reader accepts and rejects the
same archives. An archive written under revision 1 is still valid under revision 2 unless it
relied on one of the rejected behaviours (none of the three shipping writers does).

1. **Tar dialects.** Writers emit GNU tar (`ustar  \0` magic), mode `0644`, uid/gid/mtime `0`, entry
   paths longer than 100 bytes as a GNU `L` long-name record. Readers accept ustar (including the
   `prefix` field, only for POSIX `ustar\0` + `00` headers), GNU `L`/`K`, and PAX `x` (`path`, `size`)
   / `g` records. Every header checksum is verified.
2. **Entry types.** Only regular files are archive content; directory records are skipped.
   Hard links, symbolic links, devices and FIFOs make the archive invalid.
3. **Paths.** An entry path is a relative, `/`-separated, valid-UTF-8 path with no empty, `.` or
   `..` segment, no `\`, no control character, no drive prefix, at most 4096 bytes. A leading `./` is
   stripped. Anything else makes the archive invalid (`unsafe_path`).
4. **Uniqueness.** Two entries with the same path make the archive invalid. (The Rust readers keep the
   last one silently; with uniqueness enforced a hash check and a later use can never see different bytes.)
5. **Resource limits.** Readers bound compressed size, unpacked size, entry size, entry count, manifest
   size and JSON depth. Defaults: 1 GiB / 2 GiB / 1 GiB / 10 000 / 16 MiB / 64.
6. **Manifest JSON** is strict RFC 8259: valid UTF-8, no duplicate keys, no trailing data, no `\u0000`.
   `edition` (H# archives) is an optional extra string; readers ignore unknown fields but a reader that
   knows `edition` rejects an H# archive whose edition it does not support.
7. **Compat verification** (what the Rust readers do): every `CHECKSUMS.sha256` line matches its entry;
   `SIGNATURE.ed25519` verifies against a key supplied by the caller.
8. **Strict verification** (recommended before loading anything): compat verification, plus
   - every payload entry (anything but `manifest.json`, `CHECKSUMS.sha256`, `SIGNATURE.ed25519`) is
     listed in `CHECKSUMS.sha256`, once;
   - the set of `manifest.artifacts` equals the set of listed entries, and each artifact's `sha256` and
     `size` match the bytes;
   - `manifest.signature` is present iff `SIGNATURE.ed25519` is, with `algorithm` = `ed25519` and
     `signed_entry` = `CHECKSUMS.sha256`;
   - when the caller supplies a trusted key it equals `manifest.signature.public_key`.
   A signature checked only against the key embedded in the manifest proves consistency, not authenticity.
9. **Extraction** never writes outside the target directory (paths are validated at read time), replaces
   files atomically, and should target a fresh directory. Tools extract only after strict verification
   unless told otherwise.
10. **Search paths** (§8) are listed exactly: `<dir>/hlibs/` for the start directory and every ancestor;
    bit-installed libraries (`<libs>/<name>/{current,<id>}/{.bit/out,.}/`, with `<libs>` =
    `<project>/cache/libs`, `$BIT_HOME[/libs]`, `~/.hackeros/libs`); `~/.hackeros/<Language>/hlibs/`;
    `/usr/lib/HackerOS/<Language>/hlibs/` with `<Language>` ∈ `H#`, `Hacker-Lang`, `HackerScript`.

## 11. Errata to §7 (ABI) — read before relying on cross-language `.so` binding

§7 states that H# exports are "ordinary per-function typed C ABI". With H# 0.9.0 this is **not what
`h# lib build` produces**: the shared object exports Itanium-mangled C++ symbols (`_Z5f_addll`) whose
parameters are runtime types (`hs::Str`), and `headers/<name>.iface.json` records `int` / `string` as
`opaque { struct_name: "int" }` instead of `i64` / `ptr`. A HackerScript or Hacker Lang consumer therefore
cannot bind an H#-built `.so` through the synthesized-`extern` path (§6.2); only same-language `ast`
splicing (§6.1) works for H# libraries today. `.so` files built by HackerScript (plain C symbols) bind
as described. Fixing this belongs in the H# compiler (a C-ABI export attribute and correct `AbiType`
lowering), not in the container format.
