# Boot environment management

Vocabulary for managing Linux boot environments and their snapshots.

## Language

**Boot environment**:
A system environment that can be selected for booting, consisting of a root dataset and all its descendant datasets. Its members are managed together as one unit.
_Avoid_: bootenv (in prose)

**Root dataset**:
The dataset that provides a boot environment's root filesystem and defines the boundary of its membership.

**Environment container**:
A designated parent dataset whose immediate children are candidate boot environment roots. Environment containers do not overlap.

**Environment identity**:
The enduring identity of a boot environment, preserved when it is renamed. A clone is a distinct boot environment with its own identity.

**Environment name**:
A boot environment's name within its environment container. Names may repeat across containers and require qualification when ambiguous.

**Private member**:
A dataset belonging exclusively to one boot environment: its root dataset or any descendant of that root. Membership includes structural datasets that are not mounted.

**Shared dataset**:
A dataset outside every boot environment's root dataset subtree, whose data is shared across boot environments.

**Current boot environment**:
The boot environment from which the running system was booted.

**Default boot environment**:
The boot environment selected by default for booting; it need not be the current boot environment.

**Boot pool**:
The configured pool that supplies the machine's persistent default boot environment.

**Next-boot selection**:
A choice of boot environment for the next boot only, distinct from changing the persistent default.
