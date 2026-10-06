# Stock ZFSBootMenu contract

Evidence update: the maintainer subsequently supplied [target initramfs archive listings](target-initramfs-evidence.md). File presence is now known for those images; deployed ZFSBootMenu identity and behavioral acceptance remain unverified. The [installation contract](../boot-environment-installation.md) records the resulting design decisions.

Research for issue #5, 2026-10-06. This is source review, not a verified boot configuration or an implementation decision. The user confirms that the first target uses stock ZFSBootMenu binaries. The deployed version and actual target initramfs contents remain unverified.

## Selected root and default selection

ZFSBootMenu 3.1.0 constructs the final kernel command line from the **selected filesystem**, prepending its detected root prefix to that dataset name. This happens in `kexec_kernel`, after selection and before loading the target kernel/initramfs. Thus an alternate environment selected interactively supplies its own root identity, without changing pool `bootfs`. This is a source-derived conclusion about the common launch path. [3.1.0 launch implementation](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/zfsbootmenu/lib/zfsbootmenu-core.sh#L325).

The 3.1.0 manual defines these selection rules:

- Without a preference, the first pool alphabetically supplies `bootfs`.
- Pool-valued `zbm.prefer` changes pool priority.
- Dataset-valued `zbm.prefer` changes pool priority **and overrides the default environment**, if that dataset exists.
- `zbm.show` requests interactive selection.

Consequently, changing `bootfs` alone does not establish the effective default when a dataset preference is configured. [Pinned 3.1.0 manual, command-line parameters](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/man/zfsbootmenu.7.rst#L19).

Root prefixes normally use `root=zfs:`; Arch defaults to `zfs=`, Gentoo/Alpine to `root=ZFS=`. `org.zfsbootmenu:rootprefix` overrides detection. The command-line property supports recursive `%{parent}` inheritance; users should not supply their own root argument. These are separate concerns from the default selection. [Pinned 3.1.0 manual, dataset properties](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/man/zfsbootmenu.7.rst#L186).

OpenZFS 2.4.4's traditional dracut mount path uses an explicit decoded root dataset; only the `zfs:AUTO` case consults pool `bootfs`. Under systemd, this path can instead delegate root mounting to generated `sysroot.mount`. Therefore an OS integration must follow the selected root, not independently select descendants from `bootfs`. This is an implication of the explicit-root/AUTO distinction. [OpenZFS dracut mount implementation](https://github.com/openzfs/zfs/blob/zfs-2.4.4/contrib/dracut/90zfs/mount-zfs.sh.in), [systemd initramfs generator](https://github.com/openzfs/zfs/blob/zfs-2.4.4/contrib/dracut/90zfs/zfs-generator.sh.in).

## Discovery and descendants

Upstream scans filesystems with `mountpoint=/` unless `org.zfsbootmenu:active=off`; legacy roots opt in with `org.zfsbootmenu:active=on`. A scanned filesystem must itself contain a recognized kernel/initramfs pair under `/boot`. Its discovery boundary is not loka's designated container boundary. [3.1.0 upstream primer](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/general/bootenvs-and-you.rst).

The primer explicitly assigns additional filesystem mounts to the target initramfs or root filesystem. ZFSBootMenu's native snapshot cloning and rollback assume a single filesystem. **Inference:** stock ZFSBootMenu supplies root selection, but cannot itself establish loka's whole-environment mount or recovery contract. Native menu recovery operations must not be advertised as whole-environment operations for loka's nested layout. [3.1.0 upstream primer, boot environments](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/general/bootenvs-and-you.rst#L56).

## Hooks and stock binaries

External hooks can be loaded through `zbm.hookroot=device//path`, subject to filesystem-driver availability. Boot-selection hooks receive the selected dataset, kernel, initramfs, and temporary mountpoint. Menu setup hooks miss automatic boots. Boot-selection/teardown hooks cannot generally abort cleanly. These are extension points, not a supplied descendant-selection policy. [Pinned 3.1.0 hook contract](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/man/zfsbootmenu.7.rst#L282).

**Design implication:** retaining stock binaries is compatible with investigating selection in the target initramfs/OS. An external ZFSBootMenu hook is another possible integration, but requires deliberate deployment and failure behavior. No hook or mounting mechanism is selected by this note.

## Evidence still needed

Binary releases include `/etc/zbm-commit-hash` inside the ZFSBootMenu environment. Obtain the deployed image identity and its effective command line, then compare against its corresponding release source. [Official binary-release documentation](https://docs.zfsbootmenu.org/en/latest/general/binary-releases.html).

The floating `latest` manual currently describes per-kernel `.kcl` files absent from the pinned 3.1.0 manual; do not infer deployed support from that page's version heading. [Floating command-line documentation](https://docs.zfsbootmenu.org/en/latest/man/zfsbootmenu.7.html#passing-kernel-command-lines-to-boot-environments), [pinned 3.1.0 manual](https://github.com/zbm-dev/zfsbootmenu/blob/7174e420590a270ece3c6426bdb1d5cdd9ef27b0/docs/man/zfsbootmenu.7.rst).

Acceptance still needs the target initramfs's actual ZFS modules, root parser, descendant mounting path, generator timing, and cache ownership. Source inspection alone does not establish that stale cloned caches, stock ZED refreshes, or global `zfs mount -a` are isolated to the selected environment. No boot or installation mutation was performed for this note.
