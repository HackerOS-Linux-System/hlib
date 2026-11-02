# Security model

Threat: a hostile `.hlib` (or a tampered one) fed to a toolchain.

- Paths: relative only, no `..`, `\`, control chars, drive prefixes, empty segments; links/devices/FIFOs refused; duplicates refused.
- Resource limits on packed/unpacked/entry size, entry count, manifest size, JSON depth; zstd decompression is bounded.
- Tar header checksums verified; PAX/GNU long names handled.
- Strict verification: payload not covered by CHECKSUMS, manifest/archive mismatch or half-signed archives fail.
- Trust: the key in the manifest only proves consistency. Authenticity needs a key you obtained out of band (`--pubkey`, `trusted_key_hex`).
- Extraction: atomic writes, verify-before-extract in the CLI. Extract into a fresh directory.
- Secret keys are written 0600 and zeroed after use. Keep them out of CI logs.

Not covered: malicious *code* inside a verified library; a compromised signing key; side channels beyond constant-time compares.
Report vulnerabilities privately to the maintainers.
