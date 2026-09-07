# Topic 1 · Signing

*Proving who made a container image — and that nobody changed it since.*

`satisfies: SLSA L2 · SSDF "Protect Software" · ISO 27001 A.8.28`

---

## 1. What it is

When a project publishes a container image to a public registry, anyone can download it. Signing lets a downloader prove two things before trusting it:

- **Who made it** — it really came from the project, not an impostor.
- **That it's unchanged** — nobody tampered with it after it was built.

Think of a **wax seal** on a letter: the right seal proves who sent it, and an intact seal proves it wasn't opened.

### How the seal works (one idea: the hash)

A **hash** is a **fingerprint of a file** — run the file through a math function, get a short string out. Change even one bit of the file and the fingerprint comes out completely different. So the fingerprint stands for "this exact file, unaltered."

A **signature** is that fingerprint, locked by something only the real signer controls. To verify, you recompute the image's fingerprint and check that the seal covers *that* fingerprint and was made by who it claims. Match → genuine and untouched. Mismatch → rejected.

---

## 2. Keyless signing (how modern projects do it)

The old way used a **private key** — a secret file. If it's stolen, an attacker can forge signatures. A long-lived secret is a long-lived risk.

**Keyless signing removes the secret.** When the build runs in CI:

1. The CI platform (e.g. GitHub) gives the build a **short-lived identity token** proving *"I am this exact workflow, in this exact repo."* (This identity system is called **OIDC**.)
2. A service called **Fulcio** checks that token and issues a certificate valid for only **~10 minutes**, tied to that workflow's identity.
3. The image is signed with it, then the certificate **expires** — nothing is left to steal.

The permanent record of the signature is written to a public, tamper-evident log called **Rekor**, so the signature stays verifiable forever even after the certificate expires.

**The payoff:** a keyless signature carries *who signed it* (the workflow identity + the issuer). Verifying isn't just "is the seal valid?" — it's also "is it from the identity I expect?"

---

## 3. Tools

- **cosign** — the tool to sign and verify. (Part of the **sigstore** project.)
- Supporting sigstore services, used automatically: **Fulcio** (issues short-lived certs), **Rekor** (public signature log).

Install (Go toolchain):

```bash
go install github.com/sigstore/cosign/v2/cmd/cosign@latest
export PATH=$PATH:$(go env GOPATH)/bin
which cosign      # should print a path
```

---

## 4. How to verify (the plan)

Ask: *is this published image actually signed?*

```bash
cosign verify quay.io/metal3-io/cluster-api-provider-metal3:v1.13.2 \
  --certificate-identity-regexp=".*" \
  --certificate-oidc-issuer-regexp=".*" 2>&1
```

What each part means:

| Part | Meaning |
|---|---|
| `cosign verify` | "Check the seal on this image." |
| `quay.io/.../cluster-api-provider-metal3:v1.13.2` | The exact image and version to check. |
| `--certificate-identity-regexp=".*"` | *Who* to accept as signer. `.*` = **accept any identity** (first-look only). |
| `--certificate-oidc-issuer-regexp=".*"` | *Which* login system issued the signer's identity. `.*` = accept any. |
| `2>&1` | Also show error messages, so nothing is hidden. |

> **Important:** `.*` proves *a* valid signature exists — **not** that the project signed it. With `.*`, a signature from anyone's workflow would pass. To prove it's genuinely the project's, replace `.*` with the real workflow identity and issuer (see §6).

---

## 5. Result (real output, read line by line)

```
Verification for quay.io/metal3-io/cluster-api-provider-metal3:v1.13.2 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The code-signing certificate was verified using trusted certificate authority certificates
[{"critical":{"identity":{...},"image":{"docker-manifest-digest":
"sha256:b12cc1cefc86f500e9dfb03b6b565a7f1ffef5177569730873f3ae43814275a1"},
"type":"https://sigstore.dev/cosign/sign/v1"}...},
 {..."type":"https://spdx.dev/Document"...}]
```

**The three checks (the verdict):**

| Line | What it confirms |
|---|---|
| cosign claims were validated | The seal genuinely covers the image's fingerprint → **unchanged**. |
| existence in the transparency log verified | The signature is recorded in **Rekor** → can't be faked or backdated. |
| code-signing certificate verified | The signer's identity checks out against a trusted authority → **who**. |

**The JSON:**

- `docker-manifest-digest: sha256:b12cc1...` — this **is the fingerprint**. The exact identity of this image. Alter the image by one bit and this changes, breaking the signature.
- Two objects were returned. Read the `type` field on each:
  - `https://sigstore.dev/cosign/sign/v1` → **the image signature** (this topic).
  - `https://spdx.dev/Document` → **a signed SBOM** (Topic 2) — the ingredients list, attached to the same image.

**Conclusion:** the image is **signed and valid**, and it also carries a **signed SBOM** — Topics 1 and 2 confirmed on a real artifact, seen directly rather than assumed.

---

## 6. Tightening `.*` to the real identity (the real trust check)

The `.*` verify only proved *a* valid signature exists. To prove it's genuinely the project — not just *someone* — swap `.*` for the actual signing identity and issuer:

```bash
cosign verify quay.io/metal3-io/cluster-api-provider-metal3:v1.13.2 \
  --certificate-identity-regexp="https://github.com/metal3-io/.*" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" 2>&1
```

- `--certificate-identity-regexp="https://github.com/metal3-io/.*"` → only accept a signature from a **metal3-io** GitHub workflow.
- `--certificate-oidc-issuer="https://token.actions.githubusercontent.com"` → the identity must come from **GitHub Actions**, exactly — nothing else.

**Result: it still passes** — same three checks, same `b12cc1...` digest. That is the meaningful check:

| Verify | What it proves |
|---|---|
| with `.*` | a valid signature exists — could be *anyone's* |
| with real identity + issuer | the signature is **Metal3's own CI, on GitHub** — un-forgeable |

A fake image, or one signed with a stolen key from elsewhere, would **fail** the second command. Passing it walks the full keyless chain: GitHub workflow identity → Fulcio certificate → signature → Rekor log, all tying back to `metal3-io`.

*Note for later (Topic 5): `metal3-io/.*` accepts any workflow in the org. The strictest form pins the exact release workflow file. Org-level is right for proving provenance-of-signer here; exact-workflow pinning is the game when you **enforce** verification at deploy.*

---

### One-line takeaway

> Signing proves *who made an image and that it's unchanged*. Keyless does it with a short-lived identity instead of a stealable secret key — and verification only means something once you check the signer's **identity** (real issuer + identity), not just that a signature exists.
