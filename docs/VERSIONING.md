# Contract Versioning

This repository keeps immutable version snapshots under `docs/versions/`.

## Active Versions

- `v1`: original draft contracts. Preserved unchanged for compatibility and reference.
- `v2`: revised draft contracts that introduce agnostic model hosts and stricter infrastructure boundaries.

## Compatibility Rules

- A version folder must not be rewritten after another service starts implementing it.
- Additive clarifications may be made only when they do not change payload meaning.
- Breaking payload, endpoint, or responsibility changes require a new version folder.
- Top-level `docs/*.md` and `docs/openapi/*.yaml` remain the legacy v1 draft entry point until the project intentionally promotes a newer version.

## Promotion Rule

When a version becomes the implementation target for all active service repos, update the root README to mark it as current. Older versions remain available under `docs/versions/`.
