# Self-hosted Rust builds

GitHub jobs use the existing `Atleta-network` runner group. Linux work uses
`self-hosted`, `Linux`, and `X64` labels. Dynamic preflight outputs contain JSON
label arrays, including the existing old-kernel test lane and native macOS lane.
No new runner pool is provisioned. The live organization currently has Linux X64
runners only: macOS ARM64 and `oldlinux` jobs require matching capacity and must
not be executed on ordinary Linux runners to manufacture passing coverage.

## Job containers

Compiler-capable CI containers explicitly mount the installed, pinned sccache
binary and the group's trusted cache configuration. The configuration is mounted
only for non-PR events or same-repository PRs; other PR/review and issue-PR events
receive `/dev/null`. Setup masks the password before writing allowlisted values
to GitHub's environment file and gives each job its own daemon socket. Logs report
only compiler-cache counters. No credential is copied to a build context or image.

The CI image's Forklift normally replaces `RUSTC_WRAPPER`. When the trusted group
configuration is present, setup enables Forklift's supported `FORKLIFT_BYPASS=true`
mode so the unchanged `forklift cargo ...` commands invoke Cargo with sccache.
This mode exists in the image-era [Forklift source](https://github.com/paritytech/forklift/blob/a7c1a3467afb27f718b2a0460158102da4fc5be9/Commands/RootCommand.go).
Without that configuration the original Forklift path remains selected. Cargo
arguments, Rust versions, test partitions, profiles, and admission conditions are
preserved. Existing K8s, GCS, registry, signing and publishing prerequisites remain
separate requirements; this change does not create those upstream services.

The getting-started distro matrix deliberately tests fresh-user toolchain setup.
It clears an outer-runner wrapper path and keeps that fresh-user build uncached.

## Docker builders

Rust builder stages use checksum-pinned sccache 0.19.0 with a temporary BuildKit
secret, preserving their compiler arguments and runtime stages. Ordinary builds
without a secret remain usable. Standalone builds may opt in with
`--secret id=cargo_cache_env,src=/trusted/path/client.env`. Template contexts carry
their own identical installer/helper copies so their standalone context works.
The historical `Containerfile-cargo-chef` with repeated stage aliases is not used
by a checked-in GitHub workflow and remains outside this integration.

The existing srtool action and CLI launch separate reproducible-WASM containers
without an interface for this cache secret. Their original launcher and compiler
settings remain unchanged; those nested builds are not claimed to use sccache.
They need a separate reviewed launcher integration and WASM byte-equivalence
validation. The hosted-image cleanup action explicitly disables Docker image
pruning because the organization runners share a daemon.

## Reusable workflows

The sync workflows are pinned copies of `paritytech-release/sync-workflows` commit
`e179c95bcf3e46cbe6c9ebc51a0e0b26f1f931b0` with runner selectors changed. The synchronization check also corrects the
upstream undefined writer-step reference to its existing read-only reader step.
This keeps their job execution in the organization pool; a caller cannot override
the runners of a remote reusable workflow. The legacy misspelled remote
`check-syncronization.yml` path no longer exists upstream; callers now select the
vendored canonical `check-synchronization.yml`. Existing sync, permission,
release-killswitch, draft and required-job gates remain in place.

Full application image builds, actual old-kernel/macOS capacity, external-service
gates and isolated application cold/warm measurements are review gates. Shared
small-crate cache probes establish helper/backend behavior, not SDK performance.
