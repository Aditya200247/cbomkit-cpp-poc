# CBOMkit C/C++ PoC — OpenSSL Detection + OPA Policy Evaluation

> **Proof of Concept** for the LFX Mentorship 2026 Summer proposal:
> *"Extending Language and Library Support in CBOMkit — C/C++ + OpenSSL, OPA Policy Expansion, and Library Research"*
>
> Live demo → **[aditya200247.github.io/cbomkit-cpp-poc](https://aditya200247.github.io/cbomkit-cpp-poc/)**

---

## What this demonstrates

This is a single-file interactive tool that simulates the full pipeline described in the proposal — from C/C++ source scanning to CBOM generation to OPA policy evaluation. It covers every major deliverable:

| Deliverable | Where to see it |
|---|---|
| C/C++ OpenSSL detection engine | Scanner tab → select any fixture → Run Detection |
| CycloneDX 1.6 CBOM output | CBOM Output tab |
| OPA policy evaluation (7 policies) | OPA Policies tab |
| Library feasibility research | Library Research tab |
| Detection rule structure (Java + Rego) | Detection Rules tab |

---

## The five tabs

### 1. Scanner
Select one of six C/C++ fixture files from the sidebar, then click **Run Detection**. The simulated `CppDetectionEngine` scans the source, extracts cryptographic assets with file + line evidence, assigns OIDs, and flags quantum safety status — exactly what the real `OpenSslEvpCipherRules.java`, `OpenSslHashRules.java`, etc. would produce.

The six fixtures cover:

- **EVP Cipher (AES-GCM)** — `EVP_EncryptInit_ex`, `EVP_aes_256_gcm()`, `RAND_bytes`
- **EVP Digest (SHA-256)** — `EVP_DigestInit_ex`, `SHA256_Init`, `SHA1_Init` (deprecated)
- **HMAC + KDF** — `HMAC_Init_ex`, `PKCS5_PBKDF2_HMAC` with low iteration count
- **RSA + ECDSA** — `EVP_PKEY_CTX_set_rsa_keygen_bits`, `EVP_PKEY_CTX_set_ec_paramgen_curve_nid`
- **TLS / SSL Context** — `SSL_CTX_new`, `SSL_CTX_set_cipher_list`, `SSL_CTX_set_min_proto_version`
- **Weak / Deprecated** — `MD5_Init`, `DES_ecb_encrypt`, `RC4_set_key`, `EVP_des_ede3_cbc` (triggers OPA violations)

### 2. CBOM Output
Displays the generated CycloneDX 1.6 `cbom.json` with proper `cryptoProperties`, `evidence.occurrences`, and `bom-ref` fields. The "Copy JSON" button lets you grab it directly.

### 3. OPA Policies
Evaluates the CBOM against all seven proposed Rego policies and shows per-policy pass / violation / warning status with individual findings:

| Policy | What it checks |
|---|---|
| `quantum_safe.rego` | Asymmetric algorithms not on the NIST PQC whitelist |
| `weak_cipher.rego` | RC4, DES, 3DES, Blowfish, IDEA |
| `weak_hash.rego` | MD5, SHA-1 (for signatures) |
| `small_key.rego` | RSA < 2048-bit, ECC < P-256, AES < 128-bit |
| `tls_minimum.rego` | TLS version below 1.2; export-grade ciphers |
| `weak_kdf.rego` | PBKDF2 with < 100k iterations |
| `pqc_hybrid.rego` | Classical-only KEM detected; recommends ML-KEM hybrid |

### 4. Library Research
Visualizes the feasibility matrix for the five libraries named in the LFX listing — wolfSSL, libsodium, BoringSSL, Botan, and Rust crypto — with effort estimates, coverage percentages, API style, and key detection entry points.

| Library | Recommendation | Reason |
|---|---|---|
| wolfSSL | 🟢 Green | OpenSSL-compat layer means ~85% of EVP rules transfer; ~30 new `wc_*` rules needed |
| libsodium | 🟢 Green | ~60 opinionated functions; lowest effort, highest signal |
| BoringSSL | 🟡 Yellow | ~75% overlap with OpenSSL; Google-specific CBS_* / CBB_* APIs need new rules |
| Botan | 🟡 Yellow | C++ class hierarchy requires `forObjectTypes`; sonar-cxx 2.3.0 symbol table essential |
| Rust Crypto | 🔴 Red (AST) | No production sonar-rust; Cargo.toml manifest heuristic recommended as first step |

### 5. Detection Rules
Shows the actual `DetectionRuleBuilder` fluent Java API (the pattern used in `OpenSslEvpCipherRules.java`) and a sample `weak_cipher.rego` Rego policy, proving familiarity with both the sonar-cryptography codebase and the OPA integration.

---

## Best demo path (~90 seconds)

1. Sidebar → select **"Weak / Deprecated"**
2. Click **"Run Detection"** — 4 assets detected, all flagged 🔴
3. Tab → **OPA Policies** — 3 policy violations fire simultaneously
4. Tab → **CBOM Output** — inspect the CycloneDX 1.6 JSON
5. Tab → **Detection Rules** — see the `DetectionRuleBuilder` and Rego snippet

---

## Repository structure

```
cbomkit-cpp-poc/
├── index.html     ← the entire PoC (self-contained, no dependencies)
└── README.md      ← this file
```

No build step, no npm install, no server. Open `index.html` in any browser and it works.

---

## Relation to the proposal

This PoC was built as a pre-mentorship exercise to demonstrate:

- End-to-end understanding of the detection pipeline (`CppDetectionEngine` → mapper → CBOM)
- Familiarity with `DETECTION_RULE_STRUCTURE.md` and the `DetectionRuleBuilder` API
- Knowledge of the OPA integration surface in CBOMkit (`OPAComplianceService`, `opa/quantum_safe.rego`)
- Ability to reason about all six OpenSSL API categories (EVP cipher, digest, MAC, asymmetric, KDF, TLS)
- Research-level understanding of the five additional libraries named in the LFX listing

The detection rules, OPA policies, and library assessments in this PoC directly map to the deliverables described in the proposal.

---

## Related links

- Proposal: LFX Mentorship 2026 Summer — [Extending Language and Library Support in CBOMkit](https://mentorship.lfx.linuxfoundation.org/project/bdbecf7d-baaa-4adc-98a0-2ecbb4e8f05c)
- Main repo: [cbomkit/sonar-cryptography](https://github.com/cbomkit/sonar-cryptography)
- Tracking issue: [sonar-cryptography #374](https://github.com/cbomkit/sonar-cryptography/issues/374)
- Language support docs: [LANGUAGE_SUPPORT.md](https://github.com/cbomkit/sonar-cryptography/blob/main/docs/LANGUAGE_SUPPORT.md)
- Detection rule docs: [DETECTION_RULE_STRUCTURE.md](https://github.com/cbomkit/sonar-cryptography/blob/main/docs/DETECTION_RULE_STRUCTURE.md)
- OPA policy (existing): [opa/quantum_safe.rego](https://github.com/cbomkit/cbomkit/blob/main/opa/quantum_safe.rego)
