# Verifying a Calliper release

Every Calliper binary on GitHub Releases ships with four supply-chain artifacts alongside the installer. You can verify any combination of them before installing. Partial verification is meaningful; full verification is end-to-end.

---

## TL;DR: full verification on Linux

```bash
# Pick your version
VERSION=1.0.0
BASE="https://github.com/tirion-tools/calliper/releases/download/v${VERSION}"

# Download the installer + its sidecar artifacts
curl -LO "${BASE}/calliper_${VERSION}_amd64.deb"
curl -LO "${BASE}/calliper_${VERSION}_amd64.deb.sig"
curl -LO "${BASE}/calliper_${VERSION}_amd64.deb.pem"
curl -LO "${BASE}/calliper-linux.intoto.jsonl"

# Verify the cosign signature against the GitHub OIDC identity
# that signed it. Pin to the source repo and the release workflow.
cosign verify-blob \
  --certificate              "calliper_${VERSION}_amd64.deb.pem" \
  --signature                "calliper_${VERSION}_amd64.deb.sig" \
  --certificate-identity-regexp '^https://github\.com/tirion-tools/calliper-source/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer  'https://token.actions.githubusercontent.com' \
  "calliper_${VERSION}_amd64.deb"

# Verify the SLSA Build L3 provenance.
slsa-verifier verify-artifact \
  --provenance-path  calliper-linux.intoto.jsonl \
  --source-uri       github.com/tirion-tools/calliper-source \
  --source-tag       "v${VERSION}" \
  "calliper_${VERSION}_amd64.deb"

# Install
sudo dpkg -i "calliper_${VERSION}_amd64.deb"
```

If both verifications print `Verified OK`, the binary was built by the Calliper release workflow on a specific tag commit, on a GitHub-hosted runner, and hasn't been tampered with since.

---

## TL;DR: full verification on Windows

```powershell
# Pick your version
$Version = "1.0.0"
$Base    = "https://github.com/tirion-tools/calliper/releases/download/v$Version"
$Exe     = "calliper-$Version-windows-x64-setup.exe"

# Download the installer + its sidecar artifacts
Invoke-WebRequest "$Base/$Exe"                       -OutFile $Exe
Invoke-WebRequest "$Base/$Exe.sig"                   -OutFile "$Exe.sig"
Invoke-WebRequest "$Base/$Exe.pem"                   -OutFile "$Exe.pem"
Invoke-WebRequest "$Base/calliper-windows.intoto.jsonl" -OutFile "calliper-windows.intoto.jsonl"

# Verify the cosign signature against the GitHub OIDC identity
# that signed it. Pin to the source repo and the release workflow.
cosign verify-blob `
  --certificate              "$Exe.pem" `
  --signature                "$Exe.sig" `
  --certificate-identity-regexp '^https://github\.com/tirion-tools/calliper-source/\.github/workflows/release\.yml@refs/tags/v' `
  --certificate-oidc-issuer  'https://token.actions.githubusercontent.com' `
  $Exe

# Verify the SLSA Build L3 provenance.
slsa-verifier verify-artifact `
  --provenance-path  calliper-windows.intoto.jsonl `
  --source-uri       github.com/tirion-tools/calliper-source `
  --source-tag       "v$Version" `
  $Exe

# Install (SmartScreen will warn; that's expected, see below)
Start-Process $Exe
```

Same guarantee as the Linux flow: passing both verifications means the .exe was built by the named release workflow on the tagged commit and hasn't been modified since.

---

## What each artifact proves

| File | What it proves | Tool |
|---|---|---|
| `*.deb` / `*.exe` | The actual installer. | n/a |
| `*.sig` + `*.pem` | This binary was signed by an identity matching `tirion-tools/calliper-source/.github/workflows/release.yml@v*`. | [cosign](https://github.com/sigstore/cosign) |
| `*.intoto.jsonl` | SLSA Build Level 3 provenance: this binary was built by the named workflow, on a named commit, on a GitHub-hosted runner. | [slsa-verifier](https://github.com/slsa-framework/slsa-verifier) |
| `*.cdx.json` | CycloneDX SBOM: every library bundled into the installer. Read with any SBOM tool or `jq`. | [syft](https://github.com/anchore/syft) |

The cosign signature alone tells you "Tirion's CI signed this". The SLSA provenance additionally tells you "on this commit, from this workflow". Both together is what you want; the SBOM is for compliance / license review.

---

## Installing the verification tools

```bash
# Debian / Ubuntu
curl -LO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo install cosign-linux-amd64 /usr/local/bin/cosign

curl -LO https://github.com/slsa-framework/slsa-verifier/releases/latest/download/slsa-verifier-linux-amd64
sudo install slsa-verifier-linux-amd64 /usr/local/bin/slsa-verifier
```

```powershell
# Windows
winget install sigstore.cosign
# slsa-verifier: download from
# https://github.com/slsa-framework/slsa-verifier/releases/latest
# (winget doesn't have it as of writing).
```

---

## What if verification fails?

A failure here means one of:

1. **The file is not what Calliper published.** Don't install it. Re-download from the official Releases page over HTTPS.
2. **You're verifying an old version's signature against a new binary** (or vice versa). Re-download the matched sidecar files.
3. **You're checking against a forked workflow.** The cert-identity regex must match `tirion-tools/calliper-source/.github/workflows/release.yml`. If you're seeing a different identity, you have the wrong file.

If 1–3 all check out and verification still fails, please open an issue with the verification output and the SHA-256 of your downloaded binary. The Rekor transparency log ([rekor.sigstore.dev](https://rekor.sigstore.dev)) is the authoritative record. Every signature we publish lands there within seconds, immutably.

---

## Why this matters

The Calliper installer is currently **unsigned by Microsoft** (SmartScreen will warn) and **unsigned by Apple** (no macOS build yet). We'll add code-signing in a future release.

In the meantime, cosign + SLSA gives you the same guarantee as an EV cert at the supply-chain level: you can prove the binary came from Tirion's CI, on a specific commit, and hasn't been modified, without trusting any single CA.

The SmartScreen warning is expected; verify the signature and install anyway.
