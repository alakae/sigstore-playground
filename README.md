# sigstore-playground

Personal sandbox for learning keyless signing (Sigstore/cosign) and GitHub
attestations. Throwaway — no production use.

## What's here

1. `sign-test.yml` — signs a text file with `cosign sign-blob`, keyless via GitHub
   OIDC → Fulcio → Rekor. Verify with `cosign verify-blob`.
1. `sign-checksums.yml` — idiomatic multi-artifact pattern: generates `checksums.txt`
   with `sha256sum`, signs only that file, uploads artifacts + bundle. One signature
   covers all artifacts; verify with `cosign verify-blob` on `checksums.txt`, then
   `sha256sum -c checksums.txt` for the rest.
1. `sign-test-broken.yml` — same, minus `id-token: write`. Doesn't fail cleanly:
   cosign falls back to interactive device-flow OAuth and hangs instead of erroring.
1. `token-debug.yml` — dumps the raw OIDC token's claims (via
   `steve-todorov/oidc-debugger-action`) to see what actually ends up in the Fulcio
   certificate.
1. `sign-image.yml` — signs a real container image on GHCR with `cosign sign`, plus
   SLSA build provenance and an SPDX SBOM via `actions/attest`. Verify with
   `cosign verify` / `cosign tree` / `gh attestation verify`.
1. `custom-attestation.yml` — attests a blob with a fully custom predicate type
   (`https://github.com/alakae/sigstore-playground/nothing/v1`) and a hand-written
   JSON payload. Demonstrates that any arbitrary JSON can be attested; the type URI
   is just an identifier the verifier uses to interpret the content. Verify with
   `cosign verify-blob-attestation --bundle attestation.bundle --type <uri> \`
   `--certificate-identity-regexp '...' --certificate-oidc-issuer '...' artifact.txt`.
   Inspect the payload: `cat attestation.bundle | jq -r '.dsseEnvelope.payload' | base64 -d | jq .`.

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
- Sigstore signs *what was said*, not whether it's true. The signature proves "identity
  X attested this content"; the predicate fields (repo, commit, builder) are trusted
  verbatim. Platforms like `actions/attest-build-provenance` fill them from the OIDC
  token and runner context (trusted source). Hand-writing them yourself makes you the
  source of truth — fine for custom claims where you're the authority, not for provenance.
- Custom predicate types are just URIs — no schema registration needed. Pick a URI that
  you control and put any JSON you want in the payload. `--type slsaprovenance` is just
  shorthand for `https://slsa.dev/provenance/v1`.
- `--certificate-identity` matches the Subject Alternative Name (SAN) of the Fulcio cert —
  typically the workflow file + ref (branch or tag). A branch ref is a moving target; for
  immutable pinning use a tag ref or add `--certificate-github-workflow-sha` to lock to a
  specific commit.
