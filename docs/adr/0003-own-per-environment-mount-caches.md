# Own per-environment mount caches

Keep the stock host `zfs-mount-generator` and give each environment a filtered cache owned exclusively by loka, including its private members, eligible shared mounts, and external encryption-root metadata. The inspected initramfs listings do not contain that generator or its dataset cache, and external layout changes already require explicit refresh; this favors preparing persistent caches over introducing a custom root-aware generator. Disable the stock whole-pool ZED cache writer and automatic global mounting, and synchronously refresh and verify all affected environments before reporting lifecycle operations complete; the cost is maintaining offline caches and withholding supported boot selection when preparation fails.

This decision defines the implementation contract, not a claim of tested boot compatibility. See the [acceptance gates](../boot-environment-installation.md#implementation-acceptance-gates).
