# SpecterPDF

> **Steganographic file encryption** — hide any file inside a standard PDF using **AES-256-GCM** encryption. The resulting PDF opens normally in every PDF reader, but contains hidden content only recoverable with the correct password. Features a patentable **Deterministic Dynamic Decoy Synthesis** protocol to defeat brute-force attacks.

🔗 **Live Demo:** [my-stealth-pdf.vercel.app](https://my-stealth-pdf.vercel.app/)

---

## What is this, in 20 seconds?

SpecterPDF hides an encrypted file inside an ordinary PDF. The PDF opens normally in any reader — nothing looks different. Only the correct password recovers the hidden file. If an attacker brute-forces the password instead, the system doesn't fail loudly — it hands them a convincing fake file and lets them walk away thinking they won. The real file stays hidden, and they never know they were deceived.

---

## Architecture — How the Pieces Connect

```
                              ┌─────────────────────────────────────────┐
                              │            YOUR BROWSER                 │
                              │                                         │
                              │   1. Pick cover PDF + secret file       │
                              │   2. Compress the secret file           │
                              │   3. Encrypt it (AES-256-GCM)           │
                              │   4. Hide encrypted data inside PDF     │
                              │                                         │
                              │   🔒 Password & encryption key NEVER    │
                              │      leave this box                    │
                              └───────────────────┬─────────────────────┘
                                                  │
                          sends ONLY a UUID (hide)   sends ONLY a hash (wrong attempts)
                                                  │
                                                  ▼
                              ┌─────────────────────────────────────────┐
                              │         SERVER (FastAPI + MongoDB)       │
                              │                                         │
                              │   • Remembers each document's UUID      │
                              │   • Assigns a secret "patience level"   │
                              │     (3–7 wrong tries before it reacts)  │
                              │   • Tracks how many wrong attempts      │
                              │     each document has had this week     │
                              │   • If patience runs out → builds a     │
                              │     fake document on the spot and       │
                              │     sends it back                       │
                              │                                         │
                              │   🚫 Never sees: your password, your    │
                              │      encryption key, or your real file  │
                              └─────────────────────────────────────────┘
```

**In plain terms:** everything sensitive happens on your own device. The server is basically a strict bouncer — it only ever sees a random ID number and, when someone gets a password wrong, a scrambled version of that wrong guess. It physically cannot see your real password or your real file even if it wanted to.

---

## The Decoy Protocol (Patentable Mechanism)

This is the core novel mechanism. When the wrong password is entered **at or above a randomly assigned threshold** (3–7, chosen per document and never revealed to the client):

1. The browser sends `SHA-256(uuid + wrong_password)` to the server — the raw password is never transmitted.
2. The server takes the first 8 hex characters of that hash and uses it as a **32-bit cryptographic seed**.
3. That seed deterministically generates a structurally valid fake document — a fake bank statement, NDA, CSV of personnel records, JSON config, or image — entirely in memory.
4. **Deterministic mapping:** the same wrong password always returns the exact same decoy. A different wrong password returns a different decoy.
5. **Budget limiter:** the system only responds to a maximum of **10 unique wrong-password attempts per document per week**. Once that budget is exhausted, it hard-blocks with a rate-limit response rather than continuing to generate decoys.

Below the threshold, a wrong password just gets a plain "incorrect password" response — no decoy, nothing unusual. This means a legitimate user who fat-fingers their password a couple of times sees nothing suspicious. Only sustained, repeated wrong attempts — the signature of an actual brute-force attack — start triggering deception.

**The decision logic, step by step:**

```
                    Someone enters a password
                              │
                              ▼
                    Is the password correct?
                       │              │
                      YES             NO
                       │              │
                       ▼              ▼
                Real file is     Have we already seen
                 recovered        this exact wrong
                                  password before?
                                   │           │
                                  YES          NO
                                   │           │
                                   ▼           ▼
                          Send back the    Has this document run out
                          SAME fake file    of "new wrong attempts"
                          as last time      for the week (budget: 10)?
                          (consistency)        │           │
                                               YES          NO
                                                │           │
                                                ▼           ▼
                                          Block with a   Has this wrong
                                          rate-limit      attempt count
                                          message         crossed the
                                                          secret patience
                                                          level (3–7)?
                                                             │       │
                                                            NO      YES
                                                             │       │
                                                             ▼       ▼
                                                      Just say     Turn this wrong password
                                                      "wrong       into a unique number,
                                                      password"    use that number to build
                                                      — nothing    a one-of-a-kind fake
                                                      else         document, remember the
                                                                   pairing forever, send
                                                                   the fake document back
```

**Why this matters:** the fake document isn't picked from a small pre-made pile sitting on a hard drive somewhere — it's freshly assembled, in memory, specifically for that one wrong password, using a repeatable process. That's *why* the same wrong password always gets the same fake back, and a different wrong password gets a different one: the wrong password itself is the input that determines what gets built. The exact method used to turn a wrong password into a document is part of the filed patent and isn't detailed here.

The result: an attacker who keeps guessing eventually receives what looks like a successfully decrypted file. They have no reliable way to know it's fake, and no way to know whether the real file — or another hidden layer — still exists underneath.

---

## Using the Website — Step by Step

### To hide a file:

1. Go to the **live demo** and choose the **Hide** option.
2. **Upload your cover PDF** — this is the PDF that will carry the secret. It can be any normal PDF (a resume, a report, anything).
3. **Upload the secret file** you want to hide inside it — any file type.
4. **Set a password.** This is the only thing that can unlock the secret later. There's no "forgot password" option — if you lose it, the secret file is unrecoverable, by design.
5. Click **Hide / Generate**. The page will show progress as it compresses your file, encrypts it, and embeds it into the PDF.
6. **Download the resulting PDF.** This file looks and opens exactly like a normal PDF — you can email it, upload it, share it, store it anywhere. Nobody can tell just by looking at it that anything is hidden inside.

### To extract a hidden file:

1. Go to the **Extract** option on the demo.
2. **Upload the SpecterPDF** (the file you got from step 6 above, or one someone shared with you).
3. **Enter the password.**
4. Click **Extract**.
   - If the password is correct, your original secret file downloads immediately, exactly as you uploaded it.
   - If the password is wrong, you'll either see a plain "incorrect password" message, or — if enough wrong attempts have already been made on that file — you'll be served what looks like a normal downloadable file. That file is **not** the real secret; it's a decoy. There's no way to tell from the extraction page alone whether you received the real file or a decoy, which is the entire point of the protection.

### Adding a second hidden layer (optional):

1. Take a SpecterPDF you've already created (from step 6 above) and use it as the **cover PDF** in a new "Hide" operation.
2. Upload a second secret file and set a **different password**.
3. The result is a single PDF now hiding two separate secrets, each protected independently. This can be repeated to add more layers.
4. To recover everything, extract one layer at a time, starting from the outermost password and working inward.

---

1. User picks a cover PDF, a secret file, and sets a password.
2. The secret file is compressed using maximum-level compression.
3. A payload is constructed containing the filename and the compressed file data.
4. The payload is encrypted in-browser using AES-256-GCM, with the key derived via PBKDF2-SHA-256 (1,000,000 iterations, random salt and IV).
5. A unique identifier (UUID) is generated for this specific PDF and cryptographically bound into the encryption itself — meaning the encrypted data is tied to this exact document and can't be lifted out and decrypted inside a different PDF.
6. The encrypted data is injected into the PDF as a hidden object that standard PDF readers never look at or render — it's technically present in the file but invisible to any normal viewer.
7. The UUID is registered with the backend, which assigns the random threshold and starts tracking the weekly budget.
8. The modified PDF is downloaded — it opens and behaves exactly like a normal PDF.

---

## How Extraction Works

1. User uploads the SpecterPDF and enters a password.
2. The browser scans the PDF's internal structure for the hidden object tied to that document's UUID.
3. It attempts to decrypt using the entered password.
4. **Correct password** → the file decompresses, the original filename and data are recovered, and the file downloads.
5. **Wrong password** → decryption fails, which triggers the Decoy Protocol check on the backend.

---

## Recursive Layers & Nested Deniability

Because the hidden object is invisible to standard rendering, the output PDF is itself still a perfectly normal-looking carrier — which means a second secret file can be hidden inside it, with its own independent password, UUID, threshold, and decoy logic. This can be repeated multiple times.

A user with every password can peel the layers back one at a time, outermost to innermost. But an attacker who successfully brute-forces their way through one layer and triggers its decoy receives a file that looks like a complete, genuine result — with no reliable way to tell whether there's another hidden layer underneath it. Each layer's deception is independent, so compromising one layer reveals nothing about whether others exist.

---

## Crypto Design

- Two format versions exist for backward compatibility — a legacy version and the current version, auto-detected from the file itself.
- The current version cryptographically binds the encrypted data to the document's unique ID, preventing it from being copied into a different file and decrypted there.
- The password is never sent to the server, under any circumstance. Only a one-way hash of the UUID plus the wrong password is sent, and only for the purpose of tracking failed attempts.
- All encryption happens using the browser's built-in cryptography — no external crypto library dependency.
- Key derivation deliberately uses a very high iteration count, making brute-force password guessing computationally expensive.

---

## Backend Details

- Stores: UUID registry, per-UUID thresholds, weekly attempt counters, decoy assignments, and creation timestamps.
- Passwords are never stored, in any form. Only hashes of *wrong* password attempts are temporarily tracked, solely for budget enforcement.
- Rate limiting is enforced per IP address on both the hide and extract endpoints.
- Includes retry logic and timeout handling for database operations, so transient connection issues don't cause failures.
- CORS is locked down to the production frontend domain only — no open access from arbitrary origins.
- Standard security headers are enforced (content security policy, frame protection, content-type protection, referrer policy).
- Required configuration is validated on startup — the service fails immediately and visibly if critical settings are missing, rather than failing silently later.
- Decoy templates are integrity-checked on startup to confirm they're valid before the service goes live.

---

## Frontend Details

- All cryptographic operations happen client-side, in the browser — nothing sensitive is ever computed server-side.
- Input is validated before submission.
- API calls include automatic retry with backoff on transient failures.
- Progress indicators are shown during compression, encryption, and embedding.
- Drag-and-drop file upload, with a password visibility toggle for convenience.
- A maximum file size limit is enforced on both the hide and extract flows.

---

## Security Properties

- **AES-256-GCM** encryption with PBKDF2-SHA-256 key derivation at 1,000,000 iterations.
- A unique random salt and IV are generated for every single encryption operation.
- The encryption's built-in authentication check means a wrong password fails immediately and verifiably — there's no ambiguity about whether decryption succeeded.
- The server only ever stores hashes of wrong-password attempts — never anything that could reconstruct the actual password.
- **Credential isolation:** the server is never given the correct password, the encryption key, the salt, the IV, or the secret file itself. Even if the server were fully compromised, none of the protected content would be exposed.
- **Deterministic deception:** identical wrong passwords always produce identical decoys; different wrong passwords produce different decoys, so there's no inconsistency for an attacker to exploit.
- **Threshold protection:** a small number of accidental wrong attempts (typos) never triggers deception — only sustained brute-force-style guessing does.
- **Budget protection:** a weekly cap on unique wrong-password attempts prevents resource exhaustion and limits how much an attacker can probe the system.
- **Layer independence:** compromising one hidden layer has no effect on the security of any other layer.

---

## Tech Stack

**Frontend:** React, pdf-lib, pako (compression), Web Crypto API — deployed on Vercel

**Backend:** FastAPI, MongoDB Atlas — deployed on Render

---

## Security & Patent Note

The Decoy Protocol — including the deterministic decoy synthesis, randomized per-document activation threshold, weekly attempt budget, and the recursive nested-layer deniability architecture — is original work. A provisional patent application has been filed in India covering this system.

Because of the pending patent, the complete implementation source code is intentionally kept private at this stage. This document describes the system's architecture and design in detail, but is not a substitute for the full technical specification filed with the patent application.

---

## Known Limitations

- **Lost password = lost file, permanently.** There is no recovery mechanism by design — if a password is forgotten, the hidden file cannot be retrieved by anyone, including the creator.
- **Decoy documents are convincing but not exhaustive.** They're designed to pass a casual or automated check, but a determined expert manually inspecting a decoy closely enough may eventually be able to tell it apart from genuine output.
- **Very large cover or secret files** can slow down the in-browser compression and encryption step, since everything runs on the user's own device rather than a server.
- **This is a research and portfolio project, not a production security product.** It hasn't undergone formal third-party security auditing, and shouldn't be relied on for protecting anything genuinely high-stakes.
- **Layer count is currently unbounded in design but untested at scale** — nesting many layers deep hasn't been stress-tested for performance or reliability.

---

## Future Updates

- Expanding the variety and realism of decoy document types beyond the current set.
- Adding support for hiding files inside carrier formats other than PDF.
- A more guided, beginner-friendly UI flow for first-time users, including inline explanations at each step.
- Independent security review once the full patent specification is finalized and filed.
- Performance improvements for very large files and deeply nested layers.

---

Built by **Arman Khan**, final-year Computer Science student.

For questions, collaboration, or further detail beyond what's documented here, feel free to reach out.

---

## License

All rights reserved. This project and its underlying architecture are subject to a pending patent application. No part of this repository may be reproduced, reused, or implemented without permission.
