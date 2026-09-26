# 🔐 RegoLibrary Release Integrity

> **Personal engineering case study documenting my upstream contributions to Kubescape RegoLibrary's release-integrity and software supply-chain verification workflow.**

This repository documents the engineering work behind strengthening the trust boundary between **Kubescape RegoLibrary releases** and the artifacts consumed by downstream security tooling.

The work evolved from a straightforward SHA-256 checksum verification mechanism into a broader release-integrity chain involving:

- SHA-256 artifact verification
- Explicit checksum verification errors
- Downstream integrity enforcement in Kubescape
- Sigstore/Cosign keyless signing
- Signed checksum manifest verification
- Trusted-root handling
- Live trusted-root refresh
- Release rollout and compatibility
- Real GitHub Release integration testing
- Downstream consumption in Kubescape

The resulting workflow was released with **RegoLibrary v2.0.36** and subsequently consumed by Kubescape.

> **Important:** This repository is a personal engineering case study. It is not a fork or replacement implementation of RegoLibrary. The authoritative source code, project history, and releases remain in the upstream [Kubescape RegoLibrary](https://github.com/kubescape/regolibrary) repository.

---

##  The Problem

The original security question was simple:

> **How do we know that the security rules we're downloading are actually the ones we intended to download?**

RegoLibrary distributes policy and rule artifacts that are consumed by security tooling such as Kubescape.

Previously, an artifact could be downloaded and consumed without cryptographically verifying that its contents matched the expected release artifact.

That creates an artifact-integrity problem:

```text
Release Artifact
      │
      │ download
      ▼
Consumer
      │
      ▼
Policy / Rule
```

If the downloaded artifact is corrupted or replaced, the consumer needs a way to detect that before the artifact is trusted.

The first step was therefore to establish an explicit:

```text
download → verify → consume
```

boundary.

But that led to a deeper question:

> **What if the checksum manifest itself cannot be trusted?**

That question drove the second stage of the work.

---

#  Engineering Journey

The work developed incrementally rather than being implemented as one large change.

```text
SHA-256 Artifact Verification
             │
             ▼
Explicit Verification Errors
             │
             ▼
Kubescape Hard-Failure Boundary
             │
             ▼
Signed checksums.txt
             │
             ▼
Sigstore Identity Verification
             │
             ▼
Trusted Root Handling
             │
             ▼
LiveTrustedRoot
             │
             ▼
Real Release Integration Testing
             │
             ▼
RegoLibrary v2.0.36
             │
             ▼
Kubescape Consumption
```

---

# 1. Artifact Integrity — SHA-256 Checksums

### RegoLibrary #808

The first layer introduced checksum verification for release artifacts.

[RegoLibrary #808 — checksum verification](https://github.com/kubescape/regolibrary/pull/808?utm_source=chatgpt.com)

The release workflow publishes a `checksums.txt` manifest containing SHA-256 hashes for the release artifacts.

The consumer retrieves the manifest and uses it to verify downloaded artifacts.

Conceptually:

```text
GitHub Release
      │
      ├──────────────► artifact
      │
      └──────────────► checksums.txt
                              │
                              ▼
                       expected SHA-256
                              │
                              ▼
                       downloaded artifact
                              │
                              ▼
                         SHA-256 hash
                              │
                       ┌──────┴──────┐
                       │             │
                     match         mismatch
                       │             │
                       ▼             ▼
                    consume        reject
```

The important security property is that an artifact is no longer trusted merely because it was successfully downloaded.

It must first pass integrity verification.

### Verification cases covered

The implementation addressed conditions including:

- valid checksum manifests
- matching SHA-256 hashes
- checksum mismatches
- missing checksum entries
- malformed checksum manifests
- missing release artifacts
- existing release compatibility behavior

---

# 2. Verification Failures Become Explicit

### RegoLibrary #816

Checksum verification is only useful if callers can distinguish an integrity failure from an ordinary download failure.

[RegoLibrary #816 — expose checksum verification errors](https://github.com/kubescape/regolibrary/pull/816?utm_source=chatgpt.com)

The verification boundary was therefore exposed through an exported error:

```text
ErrChecksumVerification
```

The consumer can now distinguish:

```text
ordinary retrieval failure
        │
        └── existing error handling

checksum verification failure
        │
        └── security-sensitive error
```

This distinction became particularly important for Kubescape's fallback behavior.

A checksum mismatch should not be treated as an ordinary network failure that can be silently worked around.

---

# 3. Downstream Security Boundary — Kubescape

### Kubescape #3791

[Kubescape #3791 — fail on checksum verification errors](https://github.com/kubescape/kubescape/pull/3791?utm_source=chatgpt.com)

The next step moved the security boundary into the downstream consumer.

The core rule became:

```text
Checksum verification error
          │
          ▼
      hard failure
          │
          ▼
   no unverified fallback
```

This prevents a dangerous situation where an integrity failure occurs but the application silently switches to another artifact source or otherwise continues without verification.

The responsibility was intentionally divided between the two repositories.

### RegoLibrary owns

- checksum manifest parsing
- SHA-256 calculation
- checksum comparison
- classification of verification failures

### Kubescape owns

- propagating the verification error
- preventing fallback on integrity failures
- maintaining the consumer-side security boundary

This avoided duplicating checksum logic inside Kubescape.

---

# 4. The Deeper Trust Problem

Once artifact verification was working, another question became unavoidable:

> **Can we trust `checksums.txt` itself?**

A checksum proves:

```text
artifact == checksum in manifest
```

But it does **not**, by itself, prove:

```text
manifest == authentic release manifest
```

If an attacker could replace both:

```text
artifact
checksums.txt
```

they could potentially generate a new checksum for the malicious artifact.

The trust model would then become:

```text
             checksums.txt
                   │
             "trust me"
                   │
                   ▼
              SHA-256
                   │
                   ▼
              artifact
```

The checksum protects the artifact **against modification relative to the manifest**, but the manifest itself needs an authenticity mechanism.

That led to the next layer:

```text
authenticate manifest
          │
          ▼
trust checksums
          │
          ▼
verify artifact
```

---

# 5. Authenticate the Checksum Manifest

### RegoLibrary #822

[RegoLibrary #822 — sign release checksum manifest](https://github.com/kubescape/regolibrary/pull/822?utm_source=chatgpt.com)

The release process was extended to sign `checksums.txt` using **Cosign keyless signing through Sigstore**.

The release now contains:

```text
checksums.txt
checksums.sigstore.json
```

The bundle contains the information required to verify the signature and establish the identity associated with the signing operation.

The trust chain therefore becomes:

```text
GitHub Release
      │
      ├──────────────► checksums.txt
      │
      └──────────────► checksums.sigstore.json
                              │
                              ▼
                     Sigstore verification
                              │
                              ▼
                     signer identity
                              │
                              ▼
                       trusted manifest
                              │
                              ▼
                     SHA-256 verification
                              │
                              ▼
                         artifact
```

This separates two different security properties.

### Sigstore / Cosign

Establishes the authenticity and identity of the checksum manifest.

### SHA-256

Establishes the integrity of the downloaded artifact relative to that authenticated manifest.

Together:

```text
Authenticate what we trust
           ↓
Verify what we download
           ↓
Fail when verification fails
           ↓
Only then consume the artifact
```

---

# 6. Trusted-Root Handling

The next part of the work concerned how Sigstore verification material is obtained.

The initial implementation used Sigstore trusted-root fetching with local TUF caching.

That exposed an important operational concern:

> Verification should not depend on writing trusted-root material into an environment where `$HOME` may be read-only.

The implementation was adjusted to use:

```text
tuf.DefaultOptions()
        │
        ▼
DisableLocalCache = true
        │
        ▼
Fetch trusted-root material
```

This allowed verification to work in restricted execution environments without depending on a writable local cache.

---

# 7. Live Trusted Root

### RegoLibrary #826

[RegoLibrary #826 — upgrade sigstore-go and refresh trusted root](https://github.com/kubescape/regolibrary/pull/826?utm_source=chatgpt.com)

The Sigstore implementation was subsequently updated to use:

```text
LiveTrustedRoot
```

along with a newer `sigstore-go` version.

The important architectural change was moving away from treating the trusted root as a permanently fixed snapshot.

Conceptually:

```text
Consumer
   │
   ▼
LiveTrustedRoot
   │
   ├── trusted-root material
   └── refresh lifecycle
          │
          ▼
Sigstore verification
```

The verifier can therefore use refreshed trusted-root material rather than indefinitely relying on a stale snapshot.

---

# 8. The Complete Release Verification Boundary

At this point, the release flow had evolved into a layered trust chain.

```text
                    GitHub Release
                          │
              ┌───────────┴───────────┐
              │                       │
        release artifacts       checksums.txt
                                      │
                                      ▼
                           checksums.sigstore.json
                                      │
                                      ▼
                             Sigstore verification
                                      │
                                      ▼
                            signer identity check
                                      │
                                      ▼
                               Trusted root
                                      │
                                      ▼
                            Trusted checksums
                                      │
                    ┌─────────────────┘
                    ▼
             Download artifact
                    │
                    ▼
              SHA-256 hash
                    │
             ┌──────┴──────┐
             │             │
           match        mismatch
             │             │
             ▼             ▼
          consume        reject
```

The important property is that the checksum is not trusted before the manifest itself has been authenticated.

---

# 9. Release Compatibility and Rollout

Introducing signed release metadata also created a rollout concern.

Existing releases could contain:

```text
checksums.txt
```

without containing:

```text
checksums.sigstore.json
```

Therefore, the migration needed to account for the distinction between:

```text
legacy release
```

and:

```text
signed release
```

The rollout was performed so that the signed release metadata existed before downstream consumers were moved to depend on it.

This avoided introducing a consumer-side requirement for metadata that had not yet been published by the release.

---

# 10. Real Release Validation

Security-sensitive release verification should not stop at unit tests.

After the release workflow was updated, the verification path was tested against the actual published GitHub Release.

The integration test exercised the real path:

```text
GitHub Release
      │
      ▼
checksums.txt
      │
      ▼
checksums.sigstore.json
      │
      ▼
Signature verification
      │
      ▼
Signer identity verification
      │
      ▼
Trusted-root verification
      │
      ▼
SHA-256 artifact verification
      │
      ▼
Policy retrieval
```

The test was run against the published rolling `v2` release.

This validated that the individual components worked together across the actual release boundary rather than only through mocks or isolated unit tests.

---

# 11. RegoLibrary v2.0.36

The completed work was released as:

```text
RegoLibrary v2.0.36
```

The release contains the signed release-verification metadata required by the new verification path.

The resulting release model is:

```text
v2.0.36
│
├── release artifacts
├── checksums.txt
└── checksums.sigstore.json
```

The release was subsequently consumed downstream by Kubescape.

---

# 12. Downstream Kubescape Integration

### Kubescape #3893

[Kubescape #3893 — bump RegoLibrary to v2.0.36](https://github.com/kubescape/kubescape/pull/3893?utm_source=chatgpt.com)

Kubescape was updated to consume the released RegoLibrary version:

```text
RegoLibrary v2.0.36
```

This completes the downstream path:

```text
RegoLibrary Release
        │
        ▼
Signed checksum manifest
        │
        ▼
Sigstore verification
        │
        ▼
SHA-256 artifact verification
        │
        ▼
Kubescape
        │
        ▼
Security policies
```

The release-integrity work therefore isn't isolated to the library itself.

It forms part of the artifact acquisition and verification path used by a downstream CNCF security project.

---

# 🔗 The Security Chain

The complete design can be summarized as:

```text
┌──────────────────────────┐
│      GitHub Release      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│      checksums.txt       │
│      SHA-256 hashes      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  checksums.sigstore.json │
│     Sigstore bundle      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   Signature verification │
│   Signer identity check  │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│      Trusted Root        │
│     LiveTrustedRoot      │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   SHA-256 verification   │
│     Downloaded artifact  │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│      Policy retrieval    │
└──────────────────────────┘
```

---

#  Separation of Security Properties

One of the most important design decisions was keeping different security properties separate.

| Layer | Security property | Responsibility |
|---|---|---|
| `checksums.txt` | Artifact integrity reference | RegoLibrary |
| SHA-256 | Artifact integrity | RegoLibrary |
| `ErrChecksumVerification` | Explicit security failure | RegoLibrary |
| No fallback | Consumer-side enforcement | Kubescape |
| Cosign / Sigstore | Manifest authenticity | RegoLibrary |
| Signer identity | Signing identity verification | RegoLibrary |
| Trusted Root | Verification trust material | RegoLibrary |
| LiveTrustedRoot | Refreshed trust material | RegoLibrary |
| Integration test | Real release validation | RegoLibrary |
| v2.0.36 consumption | Downstream integration | Kubescape |

This prevents one mechanism from being responsible for properties it was not designed to provide.

For example:

```text
SHA-256
   ≠
manifest authenticity
```

and:

```text
Sigstore
   ≠
artifact-content hashing
```

Instead:

```text
Sigstore
   │
   └── authenticates the checksum manifest
                    │
                    ▼
               SHA-256
                    │
                    └── verifies the artifact
```

---

#  Failure Semantics

A major part of the work was deciding what should happen when verification fails.

### Valid artifact

```text
download
   ↓
verify
   ↓
match
   ↓
consume
```

### Checksum mismatch

```text
download
   ↓
verify
   ↓
mismatch
   ↓
ErrChecksumVerification
   ↓
hard failure
   ↓
NO unverified fallback
```

### Missing checksum

```text
artifact
   ↓
no checksum entry
   ↓
verification failure
```

### Malformed manifest

```text
checksums.txt
   ↓
parse failure
   ↓
verification failure
```

### Ordinary retrieval failure

The existing compatibility/fallback behavior remains separate from checksum verification failures.

This distinction is important because:

> **A security verification failure is not equivalent to an ordinary network failure.**

---

# 🔬 What I Worked On

The contribution path covered multiple layers of the release and consumption pipeline:

### RegoLibrary

- SHA-256 checksum verification
- Checksum manifest parsing
- Verification error classification
- Sigstore/Cosign release signing
- Signed bundle publication
- Sigstore verification
- Trusted-root handling
- `LiveTrustedRoot`
- `sigstore-go` upgrade
- Release compatibility
- Real-release integration validation

### Kubescape

- Propagation of checksum verification failures
- Preventing unverified fallback
- RegoLibrary version integration
- Consumption of RegoLibrary v2.0.36

---

# 📚 Upstream Contribution Index

| Project | PR | Purpose |
|---|---|---|
| RegoLibrary | #808 | SHA-256 checksum verification |
| RegoLibrary | #816 | Expose checksum verification errors |
| Kubescape | #3791 | Fail on checksum verification errors |
| RegoLibrary | #822 | Sign release checksum manifest |
| RegoLibrary | #826 | Upgrade sigstore-go and use LiveTrustedRoot |
| Kubescape | #3893 | Consume RegoLibrary v2.0.36 |

Related release-integrity work also included the release/rollout changes associated with the signed verification flow.

---

#  Engineering Lessons

## 1. Integrity is not the same as authenticity

A checksum can verify that:

```text
downloaded artifact == artifact described by manifest
```

But it does not establish that the manifest itself is authentic.

That distinction led directly to the Sigstore layer.

---

## 2. Security failures need explicit semantics

A checksum mismatch should not disappear into a generic error path.

Introducing:

```text
ErrChecksumVerification
```

allowed downstream consumers to make an explicit security decision.

---

## 3. Security boundaries should live at the correct layer

RegoLibrary knows how to verify the artifact.

Kubescape knows how to decide what to do when verification fails.

Keeping those responsibilities separate avoids duplicated security logic.

---

## 4. Fallback behavior can become a security boundary

A fallback that is perfectly reasonable for a network failure can become dangerous after an integrity failure.

Therefore:

```text
network failure
      ≠
integrity verification failure
```

---

## 5. Real release testing matters

A release verification system crosses several independent systems:

```text
GitHub Releases
        +
Sigstore
        +
TUF trusted-root material
        +
release artifacts
        +
consumer code
```

Testing only individual functions cannot fully validate those interactions.

Testing against the actual published release provided an end-to-end validation of the trust chain.

---

# 🔐 Final Trust Model

The final security model can be expressed in one sentence:

> **Authenticate the release metadata you trust, verify the artifacts you download against that authenticated metadata, fail explicitly when verification fails, and only then consume the artifact.**

Or visually:

```text
                 TRUST
                   │
                   ▼
        Authenticate manifest
                   │
                   ▼
             Trust hashes
                   │
                   ▼
          Verify artifact
                   │
                   ▼
          Handle failures
                   │
                   ▼
             Consume
```

This is the core principle behind the release-integrity work.

---

# 🚀 Result

The work evolved from a single checksum-verification problem into a layered software supply-chain trust boundary:

```text
                    RegoLibrary
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
   SHA-256          Sigstore          Trusted Root
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                         ▼
                Verified Release
                         │
                         ▼
                    Kubescape
```

The resulting workflow was released in **RegoLibrary v2.0.36** and integrated downstream into Kubescape.

The most important takeaway for me was that supply-chain security is rarely one cryptographic primitive.

It is about constructing a sequence of explicit trust decisions:

```text
What do we trust?
       ↓
How do we authenticate it?
       ↓
How do we verify what we download?
       ↓
What happens when verification fails?
       ↓
Can the downstream consumer bypass that boundary?
       ↓
Have we tested the complete path against a real release?
```

That progression—from artifact checksums to an authenticated release-verification chain—is what this case study documents.

---

## 🔗 Upstream Projects

- [Kubescape RegoLibrary](https://github.com/kubescape/regolibrary?utm_source=chatgpt.com)
- [Kubescape](https://github.com/kubescape/kubescape?utm_source=chatgpt.com)

## 📌 Disclaimer

This repository contains original documentation, diagrams, and technical notes describing my upstream contributions.

It does **not** contain a fork or replacement implementation of RegoLibrary.

The authoritative implementation, project history, releases, trademarks, and upstream assets remain subject to their respective upstream repositories and licenses.
