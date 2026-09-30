# Topic 4 · Pinning remote downloads

*Making sure a CI script runs the exact file you reviewed, every time.*

`satisfies: OpenSSF Scorecard "Pinned-Dependencies" · SLSA (hermetic, reproducible builds) · SSDF "Protect Software"`

---

## 1. The problem

CI scripts often fetch things at run time: helper scripts, tools, images.

```bash
wget -O tool https://example.com/repo/-/raw/main/tool
chmod +x tool
./tool
```

This runs **whatever is on `main` at that moment**. If the upstream changes, whether by an honest update, a mistake, or a compromised account, your CI runs something nobody reviewed. It's also not reproducible: the same commit can behave differently on different days.

## 2. The fix: pin, then verify

1. **Pin** the download to a fixed ref: a release tag or, stricter, a commit SHA.
2. **Verify** the file's sha256 before using it.
3. **Only then** execute it.

```bash
TOOL_REF="v1.2.3"
# Bumping TOOL_REF requires updating this hash
TOOL_SHA256="<64 hex chars>"

wget -O tool "https://example.com/repo/-/raw/${TOOL_REF}/tool"
echo "${TOOL_SHA256}  tool" | sha256sum -c -
chmod +x tool
./tool
```

With `set -e`, a mismatch stops the run before anything executes.

**Why both?** Tags can be moved. The pin makes intent clear; the hash is the actual guarantee.

## 3. Getting the values

```bash
# Tags and the commits they point to
git ls-remote --tags https://example.com/repo.git

# Hash the file at that ref
curl -sL -o tool "https://example.com/repo/-/raw/<ref>/tool"
sha256sum tool
```

## 4. Gotchas

- **Hashed an error page:** check `head -1 tool` shows a shebang and `wc -c tool` isn't tiny.
- **`sha256sum -c` format:** two spaces between hash and filename.
- **Maintenance:** every version bump needs a new hash. Leave a comment next to it.
- **Behaviour change:** before pinning, diff the pinned version against what CI currently pulls.

## 5. Same idea elsewhere

| Thing | Pin to |
|---|---|
| Downloaded script or binary | tag/commit + sha256 |
| Container image | digest (`image@sha256:...`), not `:latest` |
| GitHub Action | full commit SHA, not `@v4` |
| ISO / disk image | version + published checksum |

---

**CV-ready claim:** Hardened CI download paths by pinning remote scripts and artifacts to fixed refs with sha256 verification.
