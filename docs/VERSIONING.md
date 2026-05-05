# Contract Versioning

This repository keeps immutable version snapshots under `docs/versions/`.

## Active Versions

- `v1`: original draft contracts. Preserved unchanged for compatibility and reference.
- `v2`: revised draft contracts that introduce agnostic model hosts and stricter infrastructure boundaries.
- `v3`: structural reorganization of `v2`. Same architectural stance and semantic surface. Adds per-app contract docs under `docs/versions/v3/apps/`, each combining a Final Form, a Roadmap, and a Current Implementation section. URL versioning of cross-service APIs stays at `/v2/...` because the API itself has not changed.

## Compatibility Rules

- A version folder must not be rewritten after another service starts implementing it.
- Additive clarifications may be made only when they do not change payload meaning.
- Breaking payload, endpoint, or responsibility changes require a new version folder.
- Top-level `docs/*.md` and `docs/openapi/*.yaml` remain the legacy v1 draft entry point until the project intentionally promotes a newer version.

## Promotion Rule

When a version becomes the implementation target for all active service repos, update the root README to mark it as current. Older versions remain available under `docs/versions/`.
