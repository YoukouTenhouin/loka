# Boot environment lifecycle design

Confirmed contract and acceptance examples from the [issue #6 interview](https://github.com/YoukouTenhouin/loka/issues/6). The maintainer confirmed the complete design after the interview. Implementation and behavioral certification remain outstanding.

## Settled decisions

### Snapshots

- An environment snapshot captures the root and all private descendants under one shared snapshot name. Individual-dataset snapshots are outside Loka's lifecycle interface. Reject name collisions before creation.
- Live snapshots are allowed and promise crash consistency, not application consistency. Application coordination belongs to the caller.
- Users may supply snapshot names; otherwise generate unique names. Externally created snapshot groups are usable only when whole-environment capture together can be verified; matching names alone are insufficient.
- Fresh clone-source snapshots are visible, automatically named environment snapshots. Retain them until explicitly deleted, including after the clone is destroyed; no implicit cleanup or retention policy.

### Cloning

- Clone from either an explicitly selected existing environment snapshot or a fresh whole-environment snapshot of a selected environment, including the current environment.
- Allow any eligible configured destination container in the source pool, subject to existing encryption and installation constraints. Cross-pool copying is outside the clone command.
- Cloning requires a complete, verifiable source checkpoint. Refuse an incomplete source after manual changes and suggest taking a fresh snapshot; do not silently omit private members.
- Use current relative dataset paths after manual renames rather than reconstructing historical paths. Require the target snapshot on every current member and a verifiable complete checkpoint; an added child without that snapshot blocks cloning.
- Clone historical filesystem contents with the source's current applicable writable filesystem and user properties, applying Loka's required mounting and identity adjustments. Retain existing encryption dependencies. Do not promise historical property restoration or copy source bookkeeping blindly.
- If clone creation or preparation fails, retain created resources as visibly incomplete and unavailable for supported boot selection, with explicit cleanup available. Report exactly what exists. Success requires all members and required boot preparation to pass verification.

### Rollback

- Rollback restores contents in remaining datasets; it does not reconstruct historical membership or names after manual topology changes. The user is responsible for the consequences of those changes. See [the rollback scope decision](adr/0004-limit-rollback-to-remaining-datasets.md).
- For manually changed topology, leave added children without the target snapshot untouched, ignore deleted children, and roll back renamed children that retain the target snapshot. Require the root's target snapshot. Preview exactly which current datasets will roll back and which remain untouched.
- Also skip an original surviving child whose target snapshot was manually deleted. Disclose every untouched child before consent regardless of why its snapshot is missing; the root's target snapshot remains mandatory.
- External membership, name, and mount-property changes still require refresh and validation under the installation contract.
- Rollback of the running environment is allowed, so another boot environment is not a prerequisite for recovery. See [the current-environment decision](adr/0005-allow-current-environment-rollback.md). Application coordination belongs to the user; do not stop applications or reboot automatically. Successful live rollback reports that a reboot is required. Refresh and verify affected mount caches before reporting completion.
- On a rollback failure, stop immediately and report completed, failed, and untouched datasets. Do not automatically retry or reverse earlier changes; report failure even when some data has changed.

### Destructive scope and target protection

- Destruction and rollback must refuse operations that would delete another environment or an external dependent. Dependencies must be resolved explicitly first. Deleting newer history inside the target requires a precise loss preview and consent.
- Refuse rollback that requires deleting newer snapshots or bookmarks by default. Require a separate explicit choice to remove newer history, then enumerate affected datasets, discarded live changes, newer snapshots/bookmarks, and blockers before consent. Revalidate identities and dependencies immediately before mutation; a changed plan requires renewed consent. A generic force option never authorizes deleting external dependents.
- Environment destruction includes the whole subtree and its snapshots. Environment-snapshot deletion is allowed while the environment is running because it removes history, not current filesystem contents. Require a loss preview, refuse blocking holds or external clone dependencies, and never release holds or defer deletion automatically.
- Destruction of the current environment is forbidden. Environment-destruction targets and rollback targets other than the current environment must be unmounted; this does not prohibit snapshot deletion on a mounted environment. An unmounted default may be rolled back with required preparation checks; destruction of the default requires selecting another default first.
- Refuse operations requiring current-environment identification when that identification is unreliable.

### Rename

- Rename only unmounted environments within their existing container. Preserve identity, update affected caches, and preserve default selection when renaming the default.

### Promotion

- Provide an explicit whole-environment promotion command: promote every member that has a clone origin and skip members without origins. Mounted targets are allowed. Automatic promotion remains out of scope.
- Perform one promotion per member against its immediate origin, not repeated promotion until ancestry is eliminated. Report remaining ancestry; further promotion requires another explicit command.
- Preview promotion's dependency reversals and snapshot ownership transfers, including effects on other environments. Reject snapshot-name conflicts before starting; never rename conflicting snapshots automatically.
- On promotion failure, stop and report completed, failed, and untouched members without automatic reversal or retry. Refresh affected environments before reporting completion.

## Concrete lifecycle examples

### Snapshot and clone

Capture this environment under one snapshot name, `before`:

```text
rpool/ROOT/main
├── home
└── var                 (structural canmount=off)
    └── log
```

The environment snapshot consists of `main@before`, `main/home@before`, `main/var@before`, and `main/var/log@before`, captured together. Clone it to another eligible container in the same pool:

```text
rpool/TEST/trial         (new environment identity)
├── home
└── var                 (structural canmount=off)
    └── log
```

Each new member originates from its corresponding source snapshot. Contents come from `before`; applicable configuration comes from the source's current properties with required Loka adjustments. The clone is complete only after all members and boot preparation pass verification. Source snapshots remain visible and retained after `trial` is destroyed.

If `home` was externally renamed to `users`, refresh and validate first; cloning uses `trial/users` when the checkpoint remains complete and verifiable. Adding a member without `@before` blocks this clone; taking a fresh whole-environment snapshot supplies a new source.

### Rollback after manual topology changes

At capture time:

```text
rpool/ROOT/main@before
├── home@before
├── opt@before
└── scratch@before
```

After manual edits and required refresh/validation, immediately before rollback:

```text
rpool/ROOT/main          (@before retained)
├── users               (renamed home; @before retained)
├── opt                 (@before manually deleted)
└── cache               (added later; no @before)
                        (scratch and its snapshots deleted)
```

After rollback:

```text
rpool/ROOT/main          (contents restored from @before)
├── users               (contents restored from its @before; name retained)
├── opt                 (untouched)
└── cache               (untouched)
                        (scratch is not recreated)
```

Consent identifies the root and `users` as rollback targets and `opt` and `cache` as untouched. Loka does not require historical-tree reconstruction or promise to discover deleted members whose historical evidence no longer exists. If the root's `@before` is missing, refuse the operation. A live rollback also requires the user to coordinate applications and reboot afterward; switching the default is not rollback.

### Newer history and external dependents

Suppose `main@before` is followed by `main@after` and a newer bookmark, and `rpool/ROOT/trial` depends on `main@after`. Rollback to `before` must refuse while removing newer history would destroy `trial`, even when deletion of newer history was explicitly requested. A generic force option cannot widen this scope.

Once dependencies have been explicitly resolved, the destructive preview must identify the target environment and member datasets, target snapshots, live contents to be discarded, each newer snapshot/bookmark to be removed, untouched members, and blockers. Obtain consent for that plan and revalidate identities and dependencies before mutation. Changed plans require renewed consent. Do not call an API that can delete newer history or dependent clones first and validate its scope afterward.

### Rename and promotion

Renaming unmounted `rpool/ROOT/main` to `rpool/ROOT/stable` renames its subtree while preserving environment identity. Update and verify affected caches; if it was the default, preserve that selection under the new name. Renaming the running environment or moving it between containers is outside the command's scope.

For a member chain `A → B → C`, where the arrow means the right dataset is a clone depending on the left, promoting C once yields `A → C → B`. C can still depend on A. Whole-environment promotion applies one such operation to each member with an origin and skips independent members; it does not guarantee removal of all ancestry. Its preview includes transferred history and effects on other environments.

## Failure and acceptance examples

| Case | Required outcome |
| --- | --- |
| Any member already has the requested new snapshot name | Reject before snapshot creation. |
| External snapshots share a name but whole-environment capture cannot be verified | Do not accept them as a verified environment snapshot source. |
| Clone destination is in another pool | Refuse; cross-pool copying is outside the clone command. |
| Clone creates its root but a later member or cache preparation fails | Report exactly what exists, retain the incomplete result, and withhold supported boot selection; explicit cleanup remains available. |
| Rollback changes root contents but a later child fails | Stop, report failure and completed/failed/untouched members; do not automatically retry or reverse earlier changes. |
| Rollback changes data but cache verification fails | Report incomplete preparation and withhold supported boot selection until repaired; do not claim the data changes were undone. |
| Consent is followed by a changed dependency or target identity | Stop and require renewed consent for the changed plan. |
| Snapshot deletion encounters a blocking hold | Refuse; do not release the hold or silently defer deletion. |
| Environment destruction would delete an external dependent | Refuse and identify the dependency. |
| Requested destruction targets the current environment | Refuse. |
| Requested destruction targets the default | Require another default to be selected first. |
| Promotion has a snapshot-name conflict | Reject before starting; do not automatically rename snapshots. |
| A later member promotion fails after earlier members succeeded | Stop and report completed/failed/untouched members; no automatic reversal or retry. |
| Promotion succeeds but leaves an earlier ancestor | Report that ancestry; do not automatically promote again. |

## Design status and handoff

The maintainer confirmed the complete contract; no unanswered product questions remain in this interview. The glossary and ADRs record settled vocabulary and consequential rollback scope decisions. Existing installation requirements, including mount-cache verification, continue to apply.

Implementation must establish reliable evidence for snapshot grouping and completeness; matching names or creation transaction groups alone do not establish one atomic whole-environment request. If that evidence cannot be established, fail the verification rather than inventing a guarantee.

Adjacent work owns persistent-default and next-boot integration (#8), recovery machinery, locking and resource ownership (#10), exact CLI syntax and confirmation interaction (#11), and compatibility/behavioral certification (#12). Those details must preserve the observable contracts above. This interview neither implements lifecycle operations nor certifies either installation target.
