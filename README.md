# supply-chain-security-lab

*Learning software supply-chain security by verifying it on real, published software.*

---

## What this is

A hands-on study of how you **prove** a container image is trustworthy: that it came from who it claims, hasn't been tampered with, and was built the way it should have been.

Each topic is worked the same way — **understand the concept → verify it on a real image → read the actual result**. The worked example throughout is the [Metal3](https://github.com/metal3-io) project (a CNCF project with a mature build pipeline), but the concepts and commands apply to any signed container image.

This is a learning + reference repo, not a tool.

---

## How it's organised

One folder per topic. Each folder answers three questions:

```
NN-topic/
├── README.md         what it is + why it matters (the concept)
├── how-to-verify.md  the exact command, with every flag explained (the plan)
└── result.md         the real output, read line by line (the proof)
```

The rule of the repo: **nothing is asserted that wasn't verified.** Every claim has a command and a real result behind it.

---

## Topics

| # | Topic | Question it answers | Status |
|---|---|---|---|
| 1 | **Signing** | Who made it, and is it unchanged? | ✅ done |
| 2 | **SBOM** | What's inside it? | 🔜 next |
| 3 | **Provenance & SLSA** | How and where was it built? | ⬜ planned |
| 4 | **Pipeline hardening** | Is the build itself hard to attack? | ⬜ planned |
| 5 | **Verify at deploy** | Is the proof actually checked before running? | ⬜ planned |

---

## The frameworks behind it

Three industry frameworks, each answering one question:

- **SBOM** (SPDX / CycloneDX) — *what's inside?*
- **SLSA** (OpenSSF) — *how and where was it built?* (a maturity model, Build Levels 1–3)
- **NIST SSDF** (SP 800-218) — *are we developing safely?*

Rule of thumb: **SSDF is the "what," SLSA is the "how,"** and both lean on the SBOM inventory. Together they move software from *"trust me"* to *"verify me."* Each topic folder is tagged with the framework control it satisfies.

---

## Tools used

- [**cosign**](https://github.com/sigstore/cosign) — sign and verify images (part of [sigstore](https://www.sigstore.dev/))
- SBOM tooling — `bom`, Syft, Trivy *(from Topic 2)*
- [**OpenSSF Scorecard**](https://github.com/ossf/scorecard) — audit a repo's security posture *(from Topic 4)*

Getting cosign:

```bash
go install github.com/sigstore/cosign/v2/cmd/cosign@latest
export PATH=$PATH:$(go env GOPATH)/bin
```

---

## Start here

Open [`01-signing/`](01-signing/) — it verifies that a real published image is signed, and reads the output line by line. Everything else builds on it.

---

*A personal learning project. Uses public projects as examples; not affiliated with them.*
