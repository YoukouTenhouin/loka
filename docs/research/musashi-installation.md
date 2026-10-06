# Musashi installation evidence

Read-only inspection of `musashi.lan` on 2026-10-06, authorized by the user for issue #5. This encrypted machine is a required v1 acceptance target. Loka must use its existing encryption and unlock arrangement; key creation, storage, rotation, and migration are outside the agreed scope.

SSH used batch authentication and strict existing host-key verification. The local system SSH configuration had a permissions error, so inspection used `-F /dev/null`; network access required sandbox escalation. No remote configuration was modified, keys were not loaded/unloaded, and no key contents were read. `sudo -n true` reported that a password is required.

## Runtime and root selection

- openSUSE Tumbleweed `20260830`; running kernel `7.1.8-1-default`.
- `zfs version`: userspace and kernel `2.4.3-1`; RPM `zfs-2.4.3-2.6.x86_64`.
- Dracut RPM `112+suse.34.g35e16b7-1.1.x86_64`.
- Current mounted root and pool `bootfs`: `rpool/ROOT/tumbleweed`.
- Running root argument: `root=zfs:rpool/ROOT/tumbleweed`. Remaining command line: `ro quiet splash spl.spl_hostid=0x46a0cffa`.
- `org.zfsbootmenu:commandline=ro quiet splash` is local on the root dataset and inherited by descendants. `org.zfsbootmenu:keysource` and `org.zfsbootmenu:active` are unset throughout.

Evidence: `uname -r`, `/etc/os-release`, `zfs version`, `rpm -q`, `findmnt`, `/proc/cmdline`, `zpool get bootfs`, and `zfs get -r -t filesystem`.

## Dataset and encryption topology

Every listed filesystem has native ZFS `encryption=aes-256-gcm`, `encryptionroot=rpool`, `keystatus=available`, and `keyformat=passphrase`, reported with source `-`; `origin=-` throughout. Thus there is one encryption root above the environment container and no independent private-member encryption roots or existing clones in this inventory.

Only `rpool` has local `keylocation=file:///etc/zfs/rpool.key`; every other filesystem reports `keylocation=none` with source `default`. The key dependency is recorded by `encryptionroot`, not inferred from those `keylocation` values.

| Dataset | Mountpoint | Mountpoint source | canmount (all local) |
| --- | --- | --- | --- |
| `rpool` | `/` | local | off |
| `rpool/ROOT` | none | local | off |
| `rpool/ROOT/tumbleweed` | `/` | local | noauto |
| `rpool/ROOT/tumbleweed/home` | `/home` | inherited from root dataset | off |
| `rpool/ROOT/tumbleweed/home/root` | `/root` | local | noauto |
| `rpool/ROOT/tumbleweed/home/youkou` | `/home/youkou` | inherited from root dataset | noauto |
| `rpool/ROOT/tumbleweed/opt` | `/opt` | inherited from root dataset | noauto |
| `rpool/ROOT/tumbleweed/var` | `/var` | inherited from root dataset | off |
| `rpool/ROOT/tumbleweed/var/lib` | `/var/lib` | inherited from root dataset | off |
| `rpool/ROOT/tumbleweed/var/lib/docker` | `/var/lib/docker` | inherited from root dataset | noauto |
| `rpool/ROOT/tumbleweed/var/lib/flatpak` | `/var/lib/flatpak` | inherited from root dataset | noauto |
| `rpool/ROOT/tumbleweed/var/log` | `/var/log` | inherited from root dataset | noauto |
| `rpool/ROOT/tumbleweed/var/spool` | `/var/spool` | inherited from root dataset | noauto |

All eight `noauto` filesystems above were mounted at their listed paths. The root and every private descendant also had a snapshot named `dup-202609021123`; its presence is inventory evidence, not verification of a loka whole-environment operation. Evidence: `zfs list -r -o name,type,mountpoint`, `zfs get -r -o name,property,value,source encryption,encryptionroot,keystatus,keyformat,keylocation,origin,mountpoint,canmount`, and `findmnt -t zfs`.

## Actual mounting arrangement

`/usr/lib/systemd/system-generators/zfs-mount-generator` is installed. `/etc/zfs/zfs-list.cache/rpool` lists the whole inventory above with its observed mount and encryption properties. Stock ZED's history cache updater is linked at `/etc/zfs/zed.d/history_event-zfs-list-cacher.sh`; `zfs-zed.service` is enabled and active.

The generator created `opt.mount`, `home-youkou.mount`, `root.mount`, and the other private mount units under `/run/systemd/generator`. The inspected units identify the corresponding Tumbleweed datasets, name the cache as `SourcePath`, and have `After=` and `BindsTo=zfs-load-key@rpool.service`.

Crucially, `/etc/systemd/system/local-fs.target.d/localfs.conf` explicitly specifies both `Requires=` and `After=` for all seven descendant mounts: `home-youkou.mount`, `root.mount`, `opt.mount`, `var-lib-docker.mount`, `var-lib-flatpak.mount`, `var-log.mount`, and `var-spool.mount`. `systemctl show opt.mount` confirms `RequiredBy=local-fs.target` and `ActiveState=active`. This explains how the generated `noauto` units are pulled into boot. `/etc/fstab` is absent.

`zfs-mount.service` is also enabled and active; its unmodified unit runs `/usr/sbin/zfs mount -a`. **Inference:** the current `noauto` layout avoids this command mounting those private members, but the cache and fixed mount dependencies do not demonstrate correct selection between two environments with identical mountpoints. No alternate environment exists here to test that case.

The generated `zfs-load-key@rpool.service` checks `keystatus` and runs `zfs load-key rpool` only when unavailable. It has `RequiresMountsFor='/etc/zfs/rpool.key'`. Its presence verifies generated OS-side key dependency handling, not the contents of the booted initramfs.

## Initramfs evidence and remaining limits

`/etc/dracut.conf.d/zfs-keys.conf` contains `install_items+=" /etc/zfs/rpool.key "`: inclusion of the existing key is **requested**. `/boot/initrd-7.1.8-1-default` exists, mode `0600`, owned by root. Its contents were not readable during agent inspection. The maintainer subsequently supplied an [archive listing](target-initramfs-evidence.md) confirming that `etc/zfs/rpool.key` is present with mode `0600`, along with dracut ZFS hooks; no key contents were exposed. Runtime unlock behavior remains an acceptance test.

Installed `/usr/lib/dracut/modules.d/90zfs/zfs-lib.sh` decodes an explicit selected root and mounts it; its helper handles bring-up descendants only when `canmount=on` and mountpoint is `/etc`, `/bin`, `/lib`, `/lib??`, `/libx32`, or `/usr`. This installed source does not prove which copy exists in the booted image. Musashi's observed private mounts instead have the OS-side dependencies documented above.

No `zfsbootmenu` RPM or `/etc/zfsbootmenu/` directory was present. `/efi` was empty and not mounted; `/boot/efi` was absent. The deployed stock ZFSBootMenu image/version and effective bootloader command line remain unknown. No EFI filesystem was mounted for inspection. Prompt count and boot behavior after cloning still require acceptance evidence.
