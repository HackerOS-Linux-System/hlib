# API (`use "hlib" from "hlib"`)

All fallible calls return `option::Result<T, hlib_error::HlibError>`; `HlibError{code,message}`.
Codes: io, not_archive, missing_entry, checksum_mismatch, malformed_checksums, unsigned, bad_signature,
bad_key, duplicate_path, unsafe_path, limit, manifest, tar, zstd, invalid_input, not_found.

| Function | Purpose |
|---|---|
| `open(path)` / `open_with(path, limits)` / `open_bytes(buf, display, limits)` | read an archive (validates paths, types, limits) |
| `default_limits()` | 1 GiB packed / 2 GiB unpacked / 1 GiB entry / 10 000 entries / 16 MiB manifest |
| `read_manifest_json(path)` / `manifest_json(m)` | raw / rendered manifest |
| `has_entry`, `entry_paths`, `entry_bytes` | inspect contents |
| `verify_checksums(a)` | every CHECKSUMS line matches (compat rule) |
| `verify_signature(a, pubkey_hex)` | Ed25519 over CHECKSUMS bytes; key from the caller |
| `check(a, trusted_key_hex)` | `Report{ok, signed, signature_valid, trusted, problems}` |
| `check_strict(a, trusted_key_hex)` | adds: payload fully covered, manifest == archive, signature fields consistent |
| `extract_to_dir(a, dir)` / `extract_entry(a, path, dest)` | atomic writes, never outside `dir` |
| `builder_new(name, version, language)` then `add_shared_object`, `add_ast`, `add_header`, `add_bytecode`, `add_extra`, `add_file` | assemble |
| `build_bytes(b, signing_key_hex)` / `finish(b, out, signing_key_hex)` | deterministic output; `""` = unsigned |
| `generate_keypair()`, `public_key_hex(sk)`, `sha256_hex(buf)` | crypto helpers |
| `find(name, version, language, start_dir)` / `search_dirs(...)` | spec §8 lookup |
| `version()`, `spec_version()` | `1.0.0`, `1` |

Languages: `hsharp`, `hackerlang`, `hackerscript`. Set `SOURCE_DATE_EPOCH` for reproducible `created`.
