---
title: "Uncatchable stack overflow (DoS) in fulgur"
date: 2026-07-14 12:00:00 +0530
categories: [ CVE]
tags: [ fulgur]
toc: true
---

# Analyzing an Uncatchable Stack Overflow in `fulgur inspect`

While auditing Fulgur, I found a denial-of-service vulnerability in the `inspect` command. The issue is tracked as **GHSA-m58v-wpjx-9fc8** and affects `fulgur` and `fulgur-cli` versions **0.5.13 through 0.29.x`.

The vulnerable code is only a single call into `lopdf`:

```rust
pub fn inspect(path: &Path) -> crate::Result<InspectResult> {
    let doc = lopdf::Document::load(path)
        .map_err(|e| crate::Error::Other(format!("Failed to load PDF: {e}")))?;

    // ...
}
```

At first glance there's nothing unusual here. The problem is the version of `lopdf` that Fulgur depended on.

Versions of `lopdf` prior to 0.42.0 (RUSTSEC-2026-0187) recursively parse arrays and dictionaries while loading a document. There is no recursion limit, so a sufficiently deep object hierarchy eventually exhausts the process stack.

A minimal malicious document only needs to place a deeply nested array in the document catalog:

```pdf
/X [[[ ... ]]]
```

The proof of concept uses roughly 10,000 nested arrays and produces a PDF of about 21 KB.

The recursion happens during `Document::load()`. Nothing needs to inspect the object afterwards; opening the file is enough.

On an affected release:

```text
thread '<unknown>' has overflowed its stack
fatal runtime error: stack overflow, aborting
Aborted (core dumped)
exit=134
```

This is worth pointing out because the failure is not a Rust panic. The runtime aborts the process after the stack overflow, so `catch_unwind()` cannot recover from it.

The impact is narrower than many PDF parser bugs. `inspect` is intended as a developer utility rather than part of Fulgur's HTML-to-PDF pipeline. The renderer never reaches this code path, and neither the Python nor Ruby bindings expose it.

The practical attack scenarios are therefore limited to someone running `fulgur inspect` on an attacker-controlled PDF, or an application embedding `fulgur::inspect::inspect()` and passing it untrusted uploads.

The fix was straightforward. Fulgur 0.30.0 updates `lopdf` from 0.40.0 to 0.42.0, which introduces recursion-depth checks during document loading. The proof of concept no longer aborts the process after the upgrade.

Although this advisory concerns a debugging utility rather than the renderer itself, it illustrates a common issue with parser dependencies. An application may perform little more than call a library API, yet still inherit denial-of-service conditions that originate entirely inside the parser. Keeping parser libraries current is important even for tooling that is not expected to process arbitrary user input.date: 2026-07-14
authors:
  - Mitsuru
summary: >
  A crafted PDF can trigger an unrecoverable stack overflow in
  `fulgur inspect` through vulnerable versions of `lopdf`.
---

# Analyzing an Uncatchable Stack Overflow in `fulgur inspect`

While auditing Fulgur, I found a denial-of-service vulnerability in the `inspect` command. The issue is tracked as **GHSA-m58v-wpjx-9fc8** and affects `fulgur` and `fulgur-cli` versions **0.5.13 through 0.29.x`.

The vulnerable code is only a single call into `lopdf`:

```rust
pub fn inspect(path: &Path) -> crate::Result<InspectResult> {
    let doc = lopdf::Document::load(path)
        .map_err(|e| crate::Error::Other(format!("Failed to load PDF: {e}")))?;

    // ...
}
```

At first glance there's nothing unusual here. The problem is the version of `lopdf` that Fulgur depended on.

Versions of `lopdf` prior to 0.42.0 (RUSTSEC-2026-0187) recursively parse arrays and dictionaries while loading a document. There is no recursion limit, so a sufficiently deep object hierarchy eventually exhausts the process stack.

A minimal malicious document only needs to place a deeply nested array in the document catalog:

```pdf
/X [[[ ... ]]]
```

The proof of concept uses roughly 10,000 nested arrays and produces a PDF of about 21 KB.

The recursion happens during `Document::load()`. Nothing needs to inspect the object afterwards; opening the file is enough.

On an affected release:

```text
thread '<unknown>' has overflowed its stack
fatal runtime error: stack overflow, aborting
Aborted (core dumped)
exit=134
```

This is worth pointing out because the failure is not a Rust panic. The runtime aborts the process after the stack overflow, so `catch_unwind()` cannot recover from it.

The impact is narrower than many PDF parser bugs. `inspect` is intended as a developer utility rather than part of Fulgur's HTML-to-PDF pipeline. The renderer never reaches this code path, and neither the Python nor Ruby bindings expose it.

The practical attack scenarios are therefore limited to someone running `fulgur inspect` on an attacker-controlled PDF, or an application embedding `fulgur::inspect::inspect()` and passing it untrusted uploads.

The fix was straightforward. Fulgur 0.30.0 updates `lopdf` from 0.40.0 to 0.42.0, which introduces recursion-depth checks during document loading. The proof of concept no longer aborts the process after the upgrade.

Although this advisory concerns a debugging utility rather than the renderer itself, it illustrates a common issue with parser dependencies. An application may perform little more than call a library API, yet still inherit denial-of-service conditions that originate entirely inside the parser. Keeping parser libraries current is important even for tooling that is not expected to process arbitrary user input.
