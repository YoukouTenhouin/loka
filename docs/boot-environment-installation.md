# Boot environment installation contract

Confirmed contract and acceptance evidence for [issue #5](https://github.com/YoukouTenhouin/loka/issues/5). The maintainer confirmed the complete design after the interview; implementation and behavioral certification remain outstanding.

## Agreed decisions

- Discover candidate roots as immediate children of explicitly designated environment containers, such as `rpool/ROOT`; validate eligibility separately from discovery. Multiple containers may span multiple pools; reject overlapping containers and never discover an environment inside another environment.
- Use the root dataset's ZFS GUID as environment identity: rename preserves identity and clones receive new identities. Permit duplicate short names across containers; accept a short name only when unique, otherwise require the full root dataset path.
- Infer membership from the root dataset and every descendant. Shared datasets must be outside environment subtrees. See [the membership decision](adr/0001-infer-private-membership-from-dataset-subtrees.md).
- Manage all private members together. Homes inside the subtree intentionally participate in cloning and rollback.
- Include structural `canmount=off` datasets in whole-environment operations, preserving their non-mounting role. Membership and mounting are distinct.
- Require every mountable private member for successful boot. A missing member or unavailable key stops normal startup instead of allowing applications to write into an underlying directory. Structural `canmount=off` members remain unmounted.
- Support filesystem datasets only; a volume descendant makes the environment unsupported rather than being silently excluded from membership.
- Require directories needed before normal system startup, including `/usr`, `/etc`, and early system binaries/libraries, to reside in the root dataset for v1. Separate early-boot filesystem support is outside the initial contract.
- Initially target nested datasets with `zfs-mount-generator`. Legacy/fstab adaptation is manual; separate boot files are out of scope.
- Adoption requires operating-system boot integration that mounts only the selected environment's private members, including after direct selection in ZFSBootMenu. Keep the stock host `zfs-mount-generator`; each environment carries its own filtered cache maintained exclusively by loka.
- Cache input includes the selected environment's private members, eligible shared mounts, and required external encryption-root metadata. Disable the stock whole-pool ZED cache writer while retaining unrelated ZED functionality.
- Set mountable private datasets to `canmount=noauto`, preserve structural `off` members, and disable automatic global `zfs mount -a`. Explicit systemd requirements activate every selected mountable private member; shared mounting follows its existing policy through the generator. Adoption reports required changes before applying them.
- Clone, rename, rollback, and refresh must synchronously update and verify every affected environment's cache, including offline environments, before reporting success. Verify dataset references and required mount dependencies. Environments whose preparation fails remain unavailable for supported boot selection until repaired.
- Use stock ZFSBootMenu binaries for the initial target.
- Require one explicitly configured boot pool whose `bootfs` supplies the machine's persistent default. ZFSBootMenu must select that pool without a dataset-valued `zbm.prefer` override. Other pools remain manageable, but changing their `bootfs` does not change the machine's default.
- Identify the current environment from the dataset actually mounted at `/`, resolving its GUID independently of the default. In ambiguous contexts, including chroots or containers, report current state as unknown and refuse operations that require knowing it.
- Use loka for whole-environment cloning and rollback. Stock ZFSBootMenu remains supported for selecting environments; its native single-filesystem recovery operations are outside the supported whole-environment workflow.
- Require the maintainer's encrypted second target, `musashi.lan`, as a v1 acceptance case. Scope encrypted support around its verified configuration rather than promising every encryption arrangement.
- Use existing keys and installation unlock mechanisms; validate that every required private member can be unlocked and report unmet prerequisites. Key creation, credential storage, rotation, and encryption migration are outside v1.
- Initially support unencrypted environments and native ZFS encryption with one encryption root strictly above the environment container, as on Musashi. Reject independent encryption roots at environment roots or within their private descendants for v1.
- Permit external changes to dataset names, membership, and mount properties, but require explicit loka refresh and validation before booting an affected environment or performing further management operations. Do not assume cached configuration remains correct after those changes.
- Preserve existing shared-dataset mounting policies and key dependencies; reject conflicts that replace or hide a required private mount. Exclude other environments' private members even if those environments are unsupported by loka.

## Discovery, eligibility, and adoption diagnostics

Discovery reports candidate roots and their eligibility separately. A supported root has `mountpoint=/`, contains its boot files and early system paths, and heads a filesystem-only subtree with the agreed encryption topology. Required private mountpoints must be unambiguous and compatible with shared mounts; duplicate mountpoints across different environments are expected and handled by filtering.

Adoption reports dataset paths, observed values, violated requirements, and required adjustments before applying changes. It must account for the machine-wide effect of disabling global mounting and stock cache refresh: preserve shared behavior and prevent foreign private autostart, including for unsupported environments. If it cannot establish those conditions, report the required manual adjustment rather than certifying adoption or silently modifying another environment.

| Finding | Required diagnostic/result |
| --- | --- |
| Overlapping configured containers | Reject the configuration and identify the overlap. |
| Ambiguous short name | List matching full root dataset paths and require qualification. |
| Root mountpoint differs from `/` | Report the observed property and manual adjustment required; do not silently rewrite it. |
| Legacy/fstab-driven private mounting | Report unsupported initial mounting integration and manual adaptation requirements. |
| Shared data inside an environment subtree | Treat every descendant as private; require moving intended shared data outside before adoption. |
| Volume descendant or separate early-boot filesystem | Report the dataset and unsupported layout; never silently omit it from membership. |
| Unsupported encryption root placement | Identify the key dependency that falls outside the supported ancestor-root arrangement. |
| Conflicting private/shared mount | Identify the paths and datasets whose mounting would replace or hide a required member. |
| Stale cache, competing cache writer, or global mounting bypass | Report the affected environment and required refresh/integration adjustment; do not report it ready. |
| Unverifiable current/default state | Report unknown state or an installation-contract violation; refuse operations that depend on that state. |

The storage format for container/boot-pool configuration and exact validation commands are implementation choices; they do not change these boundaries.

## First target: maintainer installation

The maintainer confirmed that the inspected workspace machine is the first target. Read-only inspection on 2026-10-06 found openSUSE Tumbleweed `20260923`, ZFS userspace/kernel module `2.4.4`, dracut `112+suse.51.gf078a84`, and systemd `261.2`. These are observations, not minimum supported versions.

| Dataset | Mountpoint | `canmount` |
| --- | --- | --- |
| `rpool/ROOT/tumbleweed` | `/` | `noauto` |
| `rpool/ROOT/tumbleweed/home` | `/home` | `noauto` |
| `rpool/ROOT/tumbleweed/home/kanako` | `/home/kanako` | `on` |
| `rpool/ROOT/tumbleweed/home/youkou` | `/home/youkou` | `noauto` |

All four datasets are mounted and unencrypted. Both the mounted root and the pool's `bootfs` identify `rpool/ROOT/tumbleweed`. Its `/boot` and kernel symlink targets under `/usr/lib/modules` reside in the root dataset. The maintainer explicitly accepted private membership for both user homes.

Shared data already lives outside that subtree, including `rpool/shared_data/**`, `rpool/docker`, `rpool/share/smb`, `rpool/storage/gitea/**`, and data in `epool`.

### Examples and eligibility

- **Intended acceptance example, pending boot integration verification:** the Tumbleweed subtree above, with shared data outside it.
- **Rejected layout:** shared data beneath a boot environment root. Move that data outside the subtree before adoption if it must remain independent.
- **Manual adjustment:** `rpool/ROOT/archlinux` has a `/` root with legacy-mounted descendants.
- **Not currently eligible:** `rpool/ROOT/gentoo` uses `/mnt/gentoo` for its root mountpoint. Discovery must report this without silently changing it.

### Observed mounting configuration

The generator cache includes multiple environments in the pool. Stock ZED cache refresh is enabled. `zfs-mount.service` is also enabled and active, running global `zfs mount -a`; generator cache filtering alone therefore does not establish private-member isolation.

The home mount units are active, but only `home-kanako.mount` has automatic `local-fs.target` activation. The activation source for `/home/youkou` remains unverified. The active fstab contains only swap.

### Evidence still needed

Initial agent inspection could not read the root-only initramfs. The maintainer subsequently supplied [actual archive listings](research/target-initramfs-evidence.md): both images contain dracut ZFS root-mounting components, but neither listing contains the host `zfs-mount-generator` or its dataset cache. Embedded script contents and runtime behavior remain unverified. No deployed ZFSBootMenu version or effective configuration was established; the maintainer confirms use of stock binaries.

The [stock ZFSBootMenu research](research/stock-zfsbootmenu-contract.md) establishes selected-root handoff, default-selection precedence, and the distinction between root selection and descendant mounting. Its native snapshot recovery assumes a single filesystem and is excluded from the supported whole-environment workflow.

## Second target: musashi.lan

Authorized read-only SSH inspection on 2026-10-06 found openSUSE Tumbleweed `20260830`, kernel `7.1.8`, and OpenZFS `2.4.3`. Both the current root and pool `bootfs` identify `rpool/ROOT/tumbleweed`. See the [inspection evidence](research/musashi-installation.md) for the dataset tree, property sources, and configuration paths.

The host uses native ZFS encryption (`aes-256-gcm`), with one encryption root, `rpool`, above the environment container. Its key format is `passphrase`, its configured key location is `file:///etc/zfs/rpool.key`, and the key is available. Key contents were not read. Inspected descendants share this encryption root; no independent descendant encryption roots or clones were found.

Mountable private descendants use `canmount=noauto`; `/home`, `/var`, and `/var/lib` are non-mounting `canmount=off` containers. Systemd drop-ins explicitly require seven private mount units: `/home/youkou`, `/root`, `/opt`, `/var/lib/docker`, `/var/lib/flatpak`, `/var/log`, and `/var/spool`. The generator supplies these from an unfiltered pool cache and associates them with loading the `rpool` key. Stock ZED cache refresh and global `zfs mount -a` remain enabled.

Dracut configuration requests key-file inclusion in the target initramfs. The maintainer's subsequent [archive listing](research/target-initramfs-evidence.md) confirms `etc/zfs/rpool.key` is present with mode `0600`; actual unlocking after lifecycle operations remains to be tested. The deployed ZFSBootMenu version and effective configuration remain unverified.

This is a required acceptance target, not yet a validated supported installation. Both targets require a solution for duplicate mountpoints after cloning; existing mounting success does not establish clone isolation.

## Design status and evidence limits

The [encryption research](research/encrypted-environments.md) distinguishes environment membership from encryption-key dependencies. Musashi's topology is now known, but its actual initramfs unlock behavior still needs verification.

No further product decisions remain from the interview review. Deployed bootloader/default configuration and encrypted boot behavior remain implementation acceptance evidence, not claims established by this design interview. Completing this contract does not certify the two currently inspected installations or authorize installation changes as part of the interview.

## Implementation acceptance gates

The selected mechanism is explained in the [cache-ownership ADR](adr/0003-own-per-environment-mount-caches.md), with supporting [mounting research](research/private-member-mounting.md). Neither target is certified by documentation or archive inspection alone. Before claiming v1 support, verify:

| Case | Required result |
| --- | --- |
| Direct alternate selection in stock ZFSBootMenu | The selected root and all its mountable private members are mounted; other environments' private members are absent, including when mountpoints duplicate. |
| Default and current differ | Current follows the mounted root GUID; default follows `bootfs` in the configured boot pool. |
| Clone, rename, and rollback | Identity follows the agreed rules; caches and required dependencies name the resulting datasets, including in offline environments. |
| External relevant changes | Explicit refresh reconciles affected caches; success is withheld when verification fails. |
| Generator reload and reboot | Filtering remains effective and no stock whole-pool cache writer or automatic global mounting path bypasses it. |
| Encrypted Musashi topology | Existing credentials unlock the selected environment and required private members after lifecycle operations; external encryption-root metadata is retained. |
| Shared and structural datasets | Shared mounting/key dependencies are preserved; structural members remain unmounted but participate in whole-environment operations. |
| Missing required member or unavailable key | Normal startup stops before applications can write into underlying directories. |
| Preparation failure | The affected environment remains unavailable for supported boot selection; the operation reports the incomplete preparation and repair requirement. |
| Initramfs regeneration | Root selection still follows ZFSBootMenu's selected root; no embedded cache or hook introduces foreign-member mounting. |

Stock ZFSBootMenu recovery and dracut's kernel-option-triggered root-only snapshot/rollback paths are outside the supported whole-environment workflow. Their mere presence in an initramfs is not evidence that they run; acceptance boots must avoid enabling those conflicting recovery operations.
