# sigstore-playground

Personal sandbox for learning keyless signing (Sigstore/cosign) and GitHub
attestations. Throwaway — no production use.

## What's here

1. `sign-test.yml` — signs a text file with `cosign sign-blob`, keyless via GitHub
   OIDC → Fulcio → Rekor. Verify with `cosign verify-blob`.
1. `sign-test-broken.yml` — same, minus `id-token: write`. Doesn't fail cleanly:
   cosign falls back to interactive device-flow OAuth and hangs instead of erroring.
1. `token-debug.yml` — dumps the raw OIDC token's claims (via
   `steve-todorov/oidc-debugger-action`) to see what actually ends up in the Fulcio
   certificate.
1. `sign-image.yml` — signs a real container image on GHCR with `cosign sign`, plus
   SLSA build provenance and an SPDX SBOM via `actions/attest`. Verify with
   `cosign verify` / `cosign tree` / `gh attestation verify`.

## Things learned here worth remembering

- `cosign sign-blob` vs `cosign sign`: blobs need a `.bundle` file since there's no
  registry to push the signature into; images push it straight into the registry
  instead (no file).
- Missing `id-token: write` doesn't fail cleanly — it silently falls back to
  interactive device-flow OAuth and hangs.
- `gh attestation verify` defaults to SLSA provenance. SBOM needs
  `--predicate-type` explicit, or it's silently never checked.
- Buildx auto-attaches its own provenance unless `provenance: false` is set, which
  breaks `docker pull` and shadows the real image tag with an attestation manifest.
