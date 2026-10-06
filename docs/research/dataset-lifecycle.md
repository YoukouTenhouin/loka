# OpenZFS guarantees for nested-dataset lifecycle operations

Research for [Research OpenZFS guarantees for nested-dataset lifecycle operations](https://github.com/YoukouTenhouin/loka/issues/3), 2026-10-06. This records facts and design implications, not product decisions. No ZFS mutations or destructive experiments were performed.

## Evidence boundary

The manuals linked below describe OpenZFS `master`; implementation checks use the fixed `zfs-2.3.4` source tag. This is not a promise that every installed distribution exposes every current command flag. The eventual supported-version decision must validate its chosen baseline. Source references name functions so reviewers can locate the relevant implementation without relying on changing line numbers.

## Atomicity and operation boundaries

| Operation | Guaranteed unit and practical limitation |
| --- | --- |
| Snapshot | Recursive snapshots share a point in time. A single same-pool `lzc_snapshot` batch is all-or-nothing, including across a crash. This does not make later cloning atomic. |
| Clone | One source snapshot becomes one target dataset in the same pool. A nested boot environment needs multiple creations. Mounting after creation is another step. |
| Rollback | One dataset returns to one snapshot. Neither `-r` nor `-R` recursively rolls back descendants. Removing newer history happens before the final rollback. |
| Destroy | One snapshot batch has an atomic core operation; destruction of a hierarchy or dependency graph is a sequence, not a transaction over the whole boot environment. |
| Rename | Dataset namespace change and mount handling have distinct failure boundaries. Recursive snapshot rename uses one checked kernel sync task. |
| Promote | Promotion acts on one clone and reverses its origin relationship; a tree needs separate promotions. |

Evidence: [snapshot manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-snapshot.8.html), [clone manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-clone.8.html), [rollback manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-rollback.8.html), [`libzfs_core.c`, atomicity overview, `lzc_snapshot`, `lzc_destroy_snaps`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs_core/libzfs_core.c), [`zfs_main.c`, `destroy_callback`, `zfs_do_clone`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/cmd/zfs/zfs_main.c), [`dsl_dataset.c`, `dsl_dataset_rename_snapshot`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/module/zfs/dsl_dataset.c).

## Snapshot membership and consistency

A snapshot includes writes from system calls completed before its capture point; recursive capture covers descendants together. Application buffers and transactions spanning several system calls are outside that guarantee. Therefore application consistency requires application-specific coordination; a filesystem-consistent snapshot alone is insufficient evidence of it. This distinction is an inference from the explicitly documented capture guarantee. [Snapshot manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-snapshot.8.html)

Recursive selection follows the dataset hierarchy, not the mounted directory tree. Consequently, shared datasets nested below a selected root would be included by recursion even if product semantics exclude them; externally located datasets mounted within the root are not descendants. An explicit same-pool snapshot batch can express selected membership without sacrificing batch atomicity. Cross-pool capture has no corresponding single-batch guarantee. [Snapshot manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-snapshot.8.html), [`lzc_snapshot`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs_core/libzfs_core.c)

## Clone properties and dependencies

Cloning duplicates a snapshot's filesystem contents; it does not recursively create child datasets. `zfs_clone` validates parents, checks encryption compatibility, and calls `lzc_clone` once. Cross-pool targets fail. The CLI then mounts/shares separately: a mount failure can leave a successfully created clone. The selected API and release must determine how mounting is suppressed during construction. [`zfs_clone`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs/libzfs_dataset.c), [`zfs_do_clone`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/cmd/zfs/zfs_main.c)

Do not treat cloning as a complete source-property copy: settable properties ordinarily inherit from the target parent unless overridden, while some properties have special rules. The property policy must distinguish local, inherited, immutable, mount-related, and user properties. Encrypted clones always share their origin's key. [Property reference](https://openzfs.github.io/openzfs-docs/man/master/7/zfsprops.7.html), [clone options](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-clone.8.html)

A clone keeps its origin snapshot alive regardless of its location in the hierarchy. Promotion reverses that relationship, transfers ownership of the origin snapshot and earlier snapshots, and changes space accounting. Conflicting snapshot names and accounting constraints can prevent it; promotion is not an independent copy. [Concepts](https://openzfs.github.io/openzfs-docs/man/master/7/zfsconcepts.7.html), [promotion manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-promote.8.html)

## Destroy, holds, and bookmarks

Dataset destruction normally unmounts/unshares and rejects remaining children or clones. `-r` includes descendants; `-R` also reaches dependent clones outside the selected hierarchy. Snapshot destruction rejects holds and clones unless deferred destruction is requested. Deferred snapshots remain visible and can acquire new holds or clones. The CLI offers parsable dry-run output, but that is an observation, not a reservation against concurrent changes. [Destroy manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-destroy.8.html)

Holds prevent snapshot deletion; they do not lock a dataset namespace or prevent new writes and snapshots. Bookmarks survive removal of their source snapshot and support incremental-send history, but do not expose retained filesystem contents and are not substitutes for rollback/clone snapshots. [Hold manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-hold.8.html), [concepts](https://openzfs.github.io/openzfs-docs/man/master/7/zfsconcepts.7.html), [bookmark manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-bookmark.8.html)

## Rollback and changing topology

Normal rollback requires the latest snapshot. `-r` deletes newer snapshots/bookmarks; `-R` additionally destroys their clones. These flags affect the selected dataset's history, not each descendant's contents. [Rollback manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-rollback.8.html)

Root rollback cannot recreate a removed child dataset or remove a child created after the snapshot. This follows from each child being a separate dataset and rollback's single-dataset operation. A deleted child's destroyed snapshot supplies no restoration source. A renamed child may still have the snapshot, so membership by present-day pathname alone is insufficient. The manager must decide whether topology differences cause refusal, explicit reconciliation, or another restoration workflow. [`dsl_dataset_rollback` and its check/sync functions](https://github.com/openzfs/zfs/blob/zfs-2.3.4/module/zfs/dsl_dataset.c)

An especially important API distinction: `libzfs`'s `zfs_rollback` itself walks and destroys newer snapshots, bookmarks, and dependents before calling `lzc_rollback_to`; its boolean parameter controls force-unmount behavior, not permission to discard history. CLI policy checks are not automatically supplied to an application calling that function. Errors can arrive after earlier deletions. [`zfs_rollback`, `rollback_destroy`, `rollback_destroy_dependent`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs/libzfs_dataset.c)

## Mounted environments and rename

Linux OpenZFS supports rollback of mounted filesystems by suspending, rolling back, and resuming them. A resume error can be reported after rollback. Thus “mounted” is not a general kernel prohibition; refusing to mutate the running boot environment is a manager policy that must be specified. Busy unmounts can independently block destructive operations. [`zfs_ioc_rollback`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/module/zfs/zfs_ioctl.c), [destroy manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-destroy.8.html)

Renaming can change inherited mountpoints and cause unmount/remount operations. `-u` suppresses that handling; legacy/none mountpoints receive special treatment. `zfs_rename` performs mount preparation, a rename ioctl, and mount restoration: success of the namespace mutation does not imply success of the last phase. External strings such as fstab entries and application configuration have no automatic rewrite guarantee. [Rename manual](https://openzfs.github.io/openzfs-docs/man/master/8/zfs-rename.8.html), [`zfs_rename`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs/libzfs_dataset.c)

## Concurrency and recovery implications

Preflight queries cannot reserve later state: clone source/parent disappearance and intervening snapshot changes are explicitly considered by `libzfs`. `lzc_rollback_to` identifies the wanted snapshot and rejects incompatible changed state instead of silently choosing a new latest snapshot. A manager lock would coordinate its own processes; cooperation from external `zfs` invocations cannot be assumed. [`zfs_clone`, `zfs_rollback`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs/libzfs_dataset.c), [`lzc_rollback`](https://github.com/openzfs/zfs/blob/zfs-2.3.4/lib/libzfs_core/libzfs_core.c)

Recovery can inspect stable dataset GUIDs, snapshot creation transaction groups, origin/clones, holds, deferred-destroy state, and current mount/property state. Names can change; GUIDs retain object identity over its lifetime. These observations support reconciliation but are not an operation journal. Retrying a whole multi-step operation blindly is unsafe when earlier steps succeeded. This is a design implication of the partial-completion paths above. [Property reference](https://openzfs.github.io/openzfs-docs/man/master/7/zfsprops.7.html)

## Human decisions now precise

1. Which descendants belong to a boot environment, and how are shared datasets excluded from snapshot, clone, rollback, and destruction?
2. Does rollback require an unchanged dataset topology, and must it refuse newer history or require an explicit destructive plan? How are external clones handled?
3. Which source properties are preserved, rewritten, inherited, or rejected during cloning, especially mount and encryption properties?
4. Which running, mounted, default-boot, and externally referenced environments are protected from rename, rollback, and destroy?
5. How is incomplete construction hidden from boot selection, and what persisted evidence permits inspection, cleanup, or recovery after interruption?

These are product and safety decisions for the existing lifecycle/safety tickets. This research does not choose their answers.
