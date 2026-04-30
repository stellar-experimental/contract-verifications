# contract-verifications

This repository runs a daily build-verification of Stellar contracts and records the results.

> [!WARNING]
> This repository is an experiment. The contents should _not_ be used at this time as an input to auditing or any financial or otherwise meaningful decisions. Use this repository only to engage in the experiment and for no other purpose.

## How it works

A scheduled GitHub workflow ([`.github/workflows/verify.yml`](.github/workflows/verify.yml)) runs daily and:

1. Clones [`stellar-experimental/contract-wasms`](https://github.com/stellar-experimental/contract-wasms) and computes the sha256 of each `.wasm` in `contracts/`.
2. Skips any wasm that already has a record in `verifications/<hash>.json` or `verifications-failed/<hash>.json`.
3. For each remaining wasm, in an isolated matrix job with no token permissions:
   - Reads the wasm's contract metadata (looks up `source_repo` and `source_rev`).
   - Clones the source at that revision.
   - Runs `stellar contract build verify --wasm <local-wasm>` from inside the cloned source (from [stellar/stellar-cli#2525](https://github.com/stellar/stellar-cli/pull/2525)). Verify reads `bldimg`/`rsver`/`bldopt_*` from the wasm meta to drive the rebuild.
   - Uploads a JSON record as a workflow artifact.
4. A separate job collects the artifacts and commits them to `verifications/`.

## Record format

Successful verifications go to `verifications/<hash>.json`. Failed ones go to `verifications-failed/<hash>.json`:

```json
{
  "wasm-hash": "...",
  "build-verified": true,
  "error": null,
  "run": "https://github.com/.../actions/runs/...",
  "meta": { "...": "..." }
}
```

`build-verified` is `false` when the verify command fails (no rebuilt artifact matches, missing `source_repo`/`source_rev` meta, or the source clone failed). On failure, `error` contains the verify command output, the clone error, or a description of the missing meta. Records are written once and never re-checked — delete a record to force re-verification.
