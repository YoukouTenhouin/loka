# ZFSBootMenu and mount-generator integration contracts

Research for [Research ZFSBootMenu and zfs-mount-generator integration contracts](https://github.com/YoukouTenhouin/loka/issues/2), 2026-10-06. This records upstream facts and design implications, not decisions or an implemented compatibility promise.

## Evidence boundary

Reviewed ZFSBootMenu **3.1.0**, commit `7174e420590a270ece3c6426bdb1d5cdd9ef27b0`, and OpenZFS **2.4.4**, commit `71a9f9578616a90c3c14bb59629fb4d31bfd68d1`. GitHub's release API identified those as current releases when researched. Generator and ZED cache files were also compared with OpenZFS development commit `8de0800006459fa4f3c27e56d26969eb099d63bc`: no differences. This is source/document review; no boot, pool mutation, distro integration, or minimum-version certification was performed. Distribution patches and installed systemd/initramfs versions remain acceptance-test inputs. [OpenZFS release](https://github.com/openzfs/zfs/releases/tag/zfs-2.4.4), [ZFSBootMenu release](https://github.com/zbm-dev/zfsbootmenu/releases/tag/v3.1.0).

The ZFSBootMenu `/en/latest/` manual already describes per-kernel `.kcl` files absent from the 3.1.0 manual. Do not treat that floating documentation as a release contract; pin source and test the installed version. [3.1.0 manual][zbm-man], [development manual](https://github.com/zbm-dev/zfsbootmenu/blob/e15503228f40b3c95ded551fab86e91f3e3d230f/docs/man/zfsbootmenu.7.rst).

## Discovery, current environment, and bootability

ZFSBootMenu scans roots with `mountpoint=/` unless `org.zfsbootmenu:active=off`; legacy roots opt in with `org.zfsbootmenu:active=on`. The scanned filesystem must itself contain `/boot` with a recognized kernel/initramfs pair. Nested operating-system datasets may be mounted by the final initramfs/OS, but ZFSBootMenu's snapshot/clone/rollback UI assumes one filesystem. Its ability to boot nested layouts is not a guarantee that its own recovery operations restore the whole environment. [Upstream primer][primer], [kernel scanner][core].

**Implication:** keep kernel/initramfs files inside the root dataset, matching the agreed exclusion of separate boot files; inventory descendants separately and do not delegate whole-environment recovery to ZFSBootMenu. Shared data needs an explicit membership rule: upstream recommends separating persistent user data from coupled OS state, but has no multi-dataset environment manifest. [Upstream primer][primer].

Linux exposes actual filesystem type, source, mountpoint, and bind-mount root through `/proc/<pid>/mountinfo`, scoped to that process's mount namespace. **Proposed policy:** resolve the host `/` mount to a ZFS dataset, then cross-check dataset identity; reject ambiguous overlay, chroot, snapshot-root, or container contexts until explicitly supported. `bootfs` describes a default, not the currently mounted root; a kernel command line describes boot intent and can become stale after rename. These are deductions, not upstream identification APIs. [Linux mountinfo](https://man7.org/linux/man-pages/man5/proc_pid_mountinfo.5.html), [boot selection source][selector].

## Nested datasets and the generator: the central open decision

The generator reads cached rows and creates units named by **mountpoint**, not by environment identity. It does not choose descendants according to the running root or pool `bootfs`. Verified behavior:

- `mountpoint=legacy` or `none`, `canmount=off`, and `org.openzfs.systemd:ignore=on` suppress mount units.
- `canmount=on` entries precede `noauto` entries at the same path. Two `on` entries collide: the already-created unit remains and the later one is skipped. This is not an environment-selection policy.
- Multiple exclusively `noauto` datasets at the same path cause removal of the generated unit for that path. A unique `noauto` unit is generated without ordinary automatic activation.

These rules are documented and directly visible in the pinned generator's `line_worker` and main cache loop. [Generator manual][generator-man], [generator source, collision handling](https://github.com/openzfs/zfs/blob/71a9f9578616a90c3c14bb59629fb4d31bfd68d1/etc/systemd/system-generators/zfs-mount-generator.c#L550), [cache loop](https://github.com/openzfs/zfs/blob/71a9f9578616a90c3c14bb59629fb4d31bfd68d1/etc/systemd/system-generators/zfs-mount-generator.c#L878).

`canmount` is **not inherited**. Setting the environment root to `noauto` therefore does not protect its descendants. A root with `canmount=off` can still supply inherited mountpoint properties to mountable children. [OpenZFS properties][props].

**Required design question:** how will each boot select only its own private descendants while retaining shared datasets? A per-environment filtered cache, property switching, or an additional boot-time mechanism are candidate approaches, not endorsed upstream recipes. Any property-switching scheme must also account for a human choosing an alternate environment directly in ZFSBootMenu. Obtain the user's actual dataset properties, cache contents, and initramfs integration before choosing. Simply installing the generator does not settle this.

Systemd orders hierarchical mounts through implicit parent `Requires`/`After` dependencies; dataset hierarchy and mount hierarchy need not coincide. The generator adds import dependencies when a pool is not already imported, plus configured `org.openzfs.systemd:*` dependencies. A manual/chroot mount implementation must construct its own mount plan rather than assume generated host units apply under an arbitrary target directory. [systemd mount manual source](https://github.com/systemd/systemd/blob/main/man/systemd.mount.xml), [generator manual][generator-man].

## Cache lifecycle

`/etc/zfs/zfs-list.cache/<pool>` is the generator's early-boot input. The ZED cacher requires an existing writable file and enabled ZEDLET. It refreshes the entire pool on create/receive/import/destroy/rename and selected property changes; snapshot-only events are ignored; export empties the file. It serializes writers and avoids rewriting unchanged content. The manual explicitly calls for `systemctl daemon-reload` to rerun generators after checking output. [Cacher source][cacher], [generator manual][generator-man].

**Implications:** do not assume asynchronous ZED processing has finished when a mutation returns. Define a verifiable cache/unit refresh completion condition. An environment-local cache copied by a snapshot may be stale; offline clones cannot be assumed updated by the running system's ZED. A manually filtered cache will be replaced with the whole pool if this stock cacher owns it. Decide ownership and refresh behavior before claiming alternate-environment bootability. These follow from the paths and whole-pool enumeration in the cacher, not an upstream synchronization guarantee. [Cacher source][cacher].

## Properties and boot selection

The 3.1.0 dataset properties include `org.zfsbootmenu:active`, `kernel`, `commandline`, `rootprefix`, `keysource`, and `devicetree`. They can be inherited. Command-line `%{parent}` expansion follows the new parent chain; `kernel` falls back to the latest kernel if its selector fails. ZFSBootMenu supplies the root argument. **Implication:** record property value **and source** when planning clone/rename; flattening inherited settings or moving under a different parent can change behavior. Define a property-preservation policy instead of assuming file snapshots capture boot configuration. [3.1.0 manual][zbm-man].

Persistent default selection uses pool `bootfs`, but dataset-valued `zbm.prefer` takes precedence. Pool-valued `zbm.prefer` influences which pool supplies the default. `zbm.show`/negative timeout requires menu interaction; hooks can customize behavior. Report a configured pool default separately from an effective boot selection that cannot be verified from the running OS alone. [Selection source][selector], [3.1.0 command-line manual][zbm-man].

No built-in one-shot environment property is consumed by the reviewed 3.1.0 selection path. Thus next-boot selection is **not established as a native contract**. A specially arranged firmware one-shot entry carrying a dataset-valued `zbm.prefer`, or a custom ZFSBootMenu integration, may be feasible but requires deployment-specific state and failure/consumption semantics. Changing `bootfs` and restoring it after boot is not equivalent: failure before restoration leaves the persistent default changed. Decide whether such extra integration fits v1; no implementation is selected here. [Complete boot selector][selector], [command-line parser](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/zfsbootmenu/pre-init/zfsbootmenu-parse-commandline.sh).

## Encryption and remaining human choices

Clones share their origin's encryption key; loading/unloading an encryption root affects inheriting datasets. ZFSBootMenu may unlock via `keylocation`, prompt, or `org.zfsbootmenu:keysource`; the final OS still needs working key access. Generator key services can depend on networking or paths containing file keys. Do not infer independent key ownership or boot readiness from a successful clone. [OpenZFS encryption properties][props], [ZFSBootMenu key contract][zbm-man], [generator manual][generator-man].

Existing layout/selection decisions should settle: dataset membership and shared data; collision-free descendant selection on every boot; cache ownership and completion checks; supported distro/initramfs/version matrix; property preservation across clone/rename; encrypted-layout support; and whether next-boot selection justifies extra integration. These are now precise questions, not missing research facts. No human decision is resolved by this note.

[zbm-man]: https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/man/zfsbootmenu.7.rst
[primer]: https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/general/bootenvs-and-you.rst
[core]: https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/zfsbootmenu/lib/zfsbootmenu-core.sh
[selector]: https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/zfsbootmenu/init.d/50-import-pools
[generator-man]: https://github.com/openzfs/zfs/blob/71a9f9578616a90c3c14bb59629fb4d31bfd68d1/man/man8/zfs-mount-generator.8.in
[cacher]: https://github.com/openzfs/zfs/blob/71a9f9578616a90c3c14bb59629fb4d31bfd68d1/cmd/zed/zed.d/history_event-zfs-list-cacher.sh.in
[props]: https://github.com/openzfs/zfs/blob/71a9f9578616a90c3c14bb59629fb4d31bfd68d1/man/man7/zfsprops.7
