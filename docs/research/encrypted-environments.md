# Encrypted boot environments

Interview update: [Musashi inspection](musashi-installation.md) established the second target's native ZFS topology, and [maintainer-supplied archive evidence](target-initramfs-evidence.md) established key-file presence. The [installation contract](../boot-environment-installation.md) now requires this target for v1 and selects the shared-ancestor encryption-root profile; the original research below distinguishes facts from options before that decision.

Research for issue #5, 2026-10-06. Encryption is now a planning requirement because the user has a second encrypted target. Its layout and encryption technology have not been inspected. These findings concern **native ZFS encryption**; they do not establish that the second target uses it. Sources below are OpenZFS 2.3 manuals and floating ZFSBootMenu documentation; deployed versions still need verification.

## Constraints from upstream

`encryptionroot` identifies the dataset supplying an encrypted dataset's key. Loading its key unlocks inheriting datasets, including clones; clones share their origin's key. **Inference:** environment membership and key dependencies are different relationships. A clone placed under another container cannot be assumed to adopt that container's key. Read `encryptionroot` for each member instead of deriving it from dataset ancestry. [OpenZFS properties](https://openzfs.github.io/openzfs-docs/man/v2.3/7/zfsprops.7.html#encryptionroot).

Children ordinarily inherit encryption keys, but a child can have an independent encryption root. `load-key -r` addresses descendant encryption roots; loading a key does not mount datasets. `change-key` can change key inheritance, but clones are the documented exception to ordinary inheritance. **Inference:** do not promise that a cloned environment gets an independent passphrase, or silently rekey datasets to fit the membership tree. Unlocking one environment can also unlock other environments without mounting them. [OpenZFS key management](https://openzfs.github.io/openzfs-docs/man/v2.3/8/zfs-load-key.8.html).

Promotion reverses clone/origin dependencies and transfers snapshot ownership. That general contract does not by itself specify the encrypted lifecycle behavior loka needs to promise. **Open validation item:** exercise encryption-root changes and unlock behavior after promotion, rename, and source deletion before claiming those operations preserve bootability. [OpenZFS promotion](https://openzfs.github.io/openzfs-docs/man/v2.3/8/zfs-promote.8.html).

ZFSBootMenu unlocks encrypted roots to read their kernel and initramfs. An encrypted `/home` below an unencrypted root does not cause its bootloader prompt. Its documented interactive flow uses `keyformat=passphrase`, with a prompt fallback for inaccessible file key locations. The documented one-prompt arrangement embeds the key file in the **target OS initramfs**, stored on the encrypted root; it must remain readable only by root. Keys must not be embedded in the normally unencrypted ZFSBootMenu image. `org.zfsbootmenu:keysource` supports finding and caching keys inside ZFSBootMenu; this is not a documented automatic handoff of its loaded key state to the target OS. **Inference:** separately verify target-initramfs unlock of the selected root and OS unlock of every required private member. [ZFSBootMenu native encryption](https://docs.zfsbootmenu.org/en/latest/general/native-encryption.html).

## Design options, not decisions

- An encryption root above the environment container provides one stable shared key dependency outside environment lifecycle operations. This is a candidate initial support profile, subject to the second host's actual topology.
- An encryption root at an environment root creates clone key dependencies that can cross environment boundaries. Independent roots among private descendants add further unlock dependencies. These layouts require explicit lifecycle and boot acceptance cases; they are not equivalent to an unencrypted subtree.
- Preserve existing key ownership and unlock mechanisms initially. Supporting encrypted environments does not itself require loka to store credentials, rotate keys, or migrate encryption layouts.

## Evidence needed from the second target

Confirm native ZFS encryption versus encryption below ZFS. Obtain the dataset tree and `encryption`, `encryptionroot`, `keystatus`, `keyformat`, `keylocation`, `origin`, `mountpoint`, and `canmount` properties, including inheritance sources; key **contents** are unnecessary. Establish ZFSBootMenu/OpenZFS versions, target distribution/initramfs, and whether boot currently prompts once or twice. Verify which mechanism unlocks independently encrypted private members and what happens when their key is unavailable. No second-host inspection or installation mutation was performed.
