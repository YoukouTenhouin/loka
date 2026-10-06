# Rust and C++ integration with libzfs

Research for [Research Rust and C++ integration with libzfs](https://github.com/YoukouTenhouin/loka/issues/4), inspected 2026-10-06. This is evidence for a later language decision, not that decision. No compilation, linking, pool creation, or runtime mutation was performed.

## Finding

Both languages can call the APIs needed by this manager. C++ has the shorter direct integration path: OpenZFS ships C headers with `extern "C"` guards. Rust is practical if we accept ownership of a narrow FFI adapter and its compatibility tests. The two Rust projects examined do **not** supply a complete, maintained, direct-libzfs backend for our operations. This is a bounded survey, not a claim that no other bindings exist. [OpenZFS header][zfs] [Whamcloud generator][wham-build] [Libzetta engine][zetta-engine]

## Native API coverage

The table is checked against OpenZFS **2.3.4**, an explicit source baseline, not a proposed minimum version or claim about the latest release. Functions provide dataset operations; neither library supplies our multi-dataset boot environment transaction model. [libzfs header][zfs] [libzfs_core header][core]

| Manager capability | libzfs | libzfs_core and limitations |
|---|---|---|
| Enumerate datasets/snapshots | `zfs_iter_root`, `zfs_iter_filesystems`, `zfs_iter_snapshots` | No equivalent general enumeration API in inspected header |
| Read/write dataset properties | `zfs_prop_get`, `zfs_prop_set`, `zfs_get_user_props` | `lzc_get_props`; no general property setter in inspected header |
| Snapshot | `zfs_snapshot`, `zfs_snapshot_nvl` | `lzc_snapshot` accepts an explicit set |
| Clone | `zfs_clone` | `lzc_clone` |
| Rename | `zfs_rename` with rename flags | `lzc_rename`; lower-level semantics |
| Destroy | `zfs_destroy`, `zfs_destroy_snaps_nvl` | `lzc_destroy`, `lzc_destroy_snaps` |
| Rollback | `zfs_rollback` | `lzc_rollback`, `lzc_rollback_to` |
| Mount/unmount | `zfs_mount`, `zfs_mount_at`, `zfs_unmount` | No mounting API in inspected header |
| Pool default property | `zpool_get_prop`, `zpool_set_prop` for `bootfs` | `lzc_set_bootenv` is a different pool boot-environment data API; do not substitute it for setting `bootfs` |

`libzfs_core` is an ioctl-oriented layer with numerical errors, documented thread safety, and operation-specific atomicity. Its source still describes the interface as evolving, despite an intention to provide a committed interface. Treating its name as a blanket guarantee of a stable, complete management API would be incorrect. Multiple calls do not become a transaction simply because each underlying operation is atomic. [Core implementation and contract][core-c]

Chroot, child-process execution, bind mounts and signal cleanup remain Linux/process orchestration outside these APIs. The policy connecting pool/dataset properties to ZFSBootMenu default selection belongs to the separate boot integration decision.

## Existing Rust bindings: source-level findings

**whamcloud/rust-libzfs:** GitHub reports the repository archived; the inspected default-branch tip is `6041b5e746cdaff54552b9c53163686378f74bdf`, dated 2020-04-29. Its generated binding allowlist includes dataset opening, enumeration and property reads, but excludes the snapshot, clone, rename, rollback and mount APIs we need. Its build script hard-codes LLVM 5 and ZFS 0.7.13 include paths and additionally links `zpool`. Adopting it means substantial modernization and new API coverage, not just adding a dependency. The library code supplies wrappers with `Drop`, but that does not establish suitability of all ownership/lifetime behavior for our backend. [Repository metadata][wham-meta] [Pinned source tree][wham] [Build script][wham-build]

**Inner-Heaven/libzetta-rs:** This project has concrete recent activity, not merely an “actively developed” badge. The inspected tip `54f50ab3d6de23f946dfae4b8a8b2b2d0926598c` is a 2026-02-22 pool-import-options change; preceding default-branch changes include March and February 2025 fixes. Its manifest still identifies version 0.5.0; recent repository activity does not imply a recent crates.io release. [Commit history][zetta-history] [Manifest][zetta-manifest]

Its `DelegatingZfsEngine` delegates snapshots and snapshot destruction to `libzfs_core`, but listing, property reads and dataset destruction to subprocess-backed `ZfsOpen3`. The `ZfsEngine` trait has no clone, rename, rollback, mount/unmount or general property-write methods. Pool property setting, including modeled `bootfs`, exists through the subprocess pool implementation. Thus it can supply useful components for an explicitly mixed backend, but does not resolve the user's direct-libzfs question by itself. The manifest depends on `libzetta-zfs-core-sys` 0.5.2 and `libnv` with `nvpair`; those transitive wrappers were not separately audited. [Delegation][zetta-delegation] [Engine interface][zetta-engine] [Pool implementation][zetta-pool] [Pool properties][zetta-props] [Manifest][zetta-manifest]

## What a project-owned adapter requires

These are design implications of the C contracts, not claims that an adapter has already been proven:

- Own `libzfs_handle_t`, dataset/pool handles and allocated nvlists, pairing them with the corresponding cleanup functions. Borrowed property strings and nvlists must not outlive their owner; copy values into ordinary application types at the boundary. In Rust use private raw pointers and `Drop`; in C++ use RAII with custom deleters. Audit iterator callback ownership separately. [Handle and error APIs][zfs]
- Convert `libzfs_errno`, action and description immediately into an application error; configure library printing deliberately. For core calls preserve numerical and per-item nvlist errors. Do not collapse a partially completed multi-dataset workflow into a single success/failure flag. [libzfs][zfs] [Core][core-c]
- Start with serialized library access. Core's explicit thread-safety promise is not evidence for sharing arbitrary libzfs handles. A Rust wrapper should not assert `Send`/`Sync` without an audit; a C++ wrapper needs the same discipline. Do not unwind Rust panics or C++ exceptions through C callbacks. [Core threading contract][core-c]
- Generate Rust declarations from supported distribution headers, or expose a small C shim that hides nvlist handling and bitfield structures such as `renameflags_t`. Generated bindings reduce handwritten ABI mistakes but cannot make incompatible runtime libraries safe. Build and test against each supported OpenZFS package version. [Header][zfs]

Both approaches need native development headers and runtime libraries. OpenZFS's pkg-config file supplies libzfs/libspl include paths and links `zfs`/`nvpair`, with a dependency on `libzfs_core`. Rust adds a C-binding generation step if using bindgen, or a C compiler for a shim; C++ consumes the headers directly. Prefer distribution-provided libraries initially and do not promise a dependency-free static executable. [Upstream pkg-config template][pc]

License evidence: the examined OpenZFS files declare CDDL-1.0; Whamcloud declares MIT; Libzetta declares BSD-2-Clause. Package notices and any redistributed native code or generated/copied material need review when choosing the actual dependency set. These facts do not establish a legal conclusion about a final distribution. [OpenZFS][zfs] [Whamcloud license][wham-license] [Libzetta manifest][zetta-manifest]

## Decision inputs and remaining questions

| Option | Main benefit | Cost still owned by this project |
|---|---|---|
| C++ with libzfs/core | Direct use of upstream headers | RAII/error adapter, version compatibility and workflow safety |
| Rust with narrow FFI or C shim | Safe application-facing ownership boundary | Binding/shim maintenance and audited unsafe code, plus the same compatibility/workflow work |
| Rust using surveyed crates | Some reusable wrappers | Missing operation coverage; archived code or explicit subprocess dependence |
| Explicit `zfs`/`zpool` subprocess backend | Avoids linking to library ABI | Output/error parsing, subprocess lifecycle and command-version compatibility; Libzetta demonstrates this tradeoff |

Confidence is high in the inspected source/API coverage and maintenance observations; build feasibility and target-distribution compatibility remain unverified. The next language decision should settle: (1) willingness to maintain a small FFI/shim layer, (2) supported distributions/OpenZFS versions, and (3) whether any subprocess backend is acceptable. If direct Rust integration remains the preferred route, a later focused build check should cover enumeration, user-property reads/writes, nvlist creation and callback cleanup against the chosen distro packages before implementation is committed to it.

[zfs]: https://github.com/openzfs/zfs/blob/zfs-2.3.4/include/libzfs.h
[core]: https://github.com/openzfs/zfs/blob/zfs-2.3.4/include/libzfs_core.h
[core-c]: https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs_core/libzfs_core.c
[pc]: https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs/libzfs.pc.in
[wham-meta]: https://api.github.com/repos/whamcloud/rust-libzfs
[wham]: https://github.com/whamcloud/rust-libzfs/tree/6041b5e746cdaff54552b9c53163686378f74bdf
[wham-build]: https://github.com/whamcloud/rust-libzfs/blob/6041b5e746cdaff54552b9c53163686378f74bdf/libzfs-sys/build.rs
[wham-license]: https://github.com/whamcloud/rust-libzfs/blob/6041b5e746cdaff54552b9c53163686378f74bdf/LICENSE
[zetta-history]: https://github.com/Inner-Heaven/libzetta-rs/commits/54f50ab3d6de23f946dfae4b8a8b2b2d0926598c
[zetta-manifest]: https://github.com/Inner-Heaven/libzetta-rs/blob/54f50ab3d6de23f946dfae4b8a8b2b2d0926598c/Cargo.toml
[zetta-engine]: https://github.com/Inner-Heaven/libzetta-rs/blob/54f50ab3d6de23f946dfae4b8a8b2b2d0926598c/src/zfs/mod.rs
[zetta-delegation]: https://github.com/Inner-Heaven/libzetta-rs/blob/54f50ab3d6de23f946dfae4b8a8b2b2d0926598c/src/zfs/delegating.rs
[zetta-pool]: https://github.com/Inner-Heaven/libzetta-rs/blob/54f50ab3d6de23f946dfae4b8a8b2b2d0926598c/src/zpool/open3.rs
[zetta-props]: https://github.com/Inner-Heaven/libzetta-rs/blob/54f50ab3d6de23f946dfae4b8a8b2b2d0926598c/src/zpool/properties.rs
