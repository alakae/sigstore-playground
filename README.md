# sigstore-playground

Personal sandbox for learning keyless signing (Sigstore/cosign) and GitHub
attestations. Throwaway — no production use.

## What's here

Seven workflows, each a self-contained sign/attest -> verify demo, ordered below
by increasing complexity. The matrix shows what each signing workflow produces;
one-liners and the two diagnostic workflows follow.

| Workflow | Applies to | Signature | Attestation | SLSA provenance | SBOM | Custom predicate |
|----------|------------|:---------:|:-----------:|:---------------:|:----:|:----------------:|
| [`sign-test`](.github/workflows/sign-test.yml) | file | ✓ | | | | |
| [`sign-checksums`](.github/workflows/sign-checksums.yml) | files (via `checksums.txt`) | ✓ | | | | |
| [`attest-blob-provenance`](.github/workflows/attest-blob-provenance.yml) | file | | ✓ | ✓ | | |
| [`custom-attestation`](.github/workflows/custom-attestation.yml) | file | | ✓ | | | ✓ |
| [`sign-image`](.github/workflows/sign-image.yml) | image | ✓ | ✓ | ✓ | ✓ | |

- [`sign-test`](.github/workflows/sign-test.yml) — minimal keyless loop:
  `cosign sign-blob` on a text file (GitHub OIDC → Fulcio → Rekor), verified with
  `cosign verify-blob`.
- [`sign-checksums`](.github/workflows/sign-checksums.yml) — idiomatic
  multi-artifact pattern: sign one `checksums.txt`, then `sha256sum -c` enforces
  the artifacts against the signed manifest.
- [`custom-attestation`](.github/workflows/custom-attestation.yml) — attests a
  blob with a fully custom predicate type URI and a hand-written JSON payload;
  the type URI is just an identifier the verifier uses to interpret the content.
- [`attest-blob-provenance`](.github/workflows/attest-blob-provenance.yml) — SLSA
  build provenance on a plain file, showing `actions/attest` is not image-specific
  (`subject-path`).
- [`sign-image`](.github/workflows/sign-image.yml) — signs a real GHCR container
  image with `cosign sign`, plus SLSA build provenance and an SPDX SBOM via
  `actions/attest` (stored in the GitHub Attestations API as well as Rekor).

### Diagnostics / anti-patterns

- [`sign-test-broken`](.github/workflows/sign-test-broken.yml) — same as
  `sign-test` but without `id-token: write`. Doesn't fail cleanly: cosign falls
  back to interactive device-flow OAuth and hangs.
- [`token-debug`](.github/workflows/token-debug.yml) — dumps the raw OIDC token
  claims (via `steve-todorov/oidc-debugger-action`) to see what lands in the
  Fulcio certificate.

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
- For GitHub Actions, the Fulcio cert extensions already answer most "how was it built"
  questions (workflow, repo, SHA, trigger, runner). SLSA provenance adds value on top by
  expressing *what went in* (resolved dependencies, source digest, external parameters) in
  a build-system-agnostic schema — most relevant in heterogeneous environments with multiple
  CI systems or third-party builders where a common format and policy evaluation is needed.
- `gh attestation verify` and `cosign verify-blob-attestation` are equivalent — the former
  is just a convenience wrapper. They differ only in retrieval: `actions/attest` stores the
  bundle in GitHub's Attestations API (fetch with `gh attestation download --repo <repo> --digest sha256:<digest>`);
  `cosign attest-blob` writes to the public Rekor log and produces a local `.bundle` file.
  Once you have the bundle, `cosign verify-blob-attestation --bundle` works for both.
- `actions/attest` works on any artifact, not just images — use `subject-path` for local
  files instead of `subject-name`/`subject-digest`.
- `--certificate-identity` matches the Subject Alternative Name (SAN) of the Fulcio cert —
  typically the workflow file + ref (branch or tag). A branch ref is a moving target; for
  immutable pinning use a tag ref or add `--certificate-github-workflow-sha` to lock to a
  specific commit.
