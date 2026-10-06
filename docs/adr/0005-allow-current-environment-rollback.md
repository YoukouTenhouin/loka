# Allow current-environment rollback

Allow rollback of the running boot environment so recovery does not require an alternative boot environment. Requiring an offline target would make this workflow impractical when no alternative exists, since stock ZFSBootMenu rollback is outside the supported whole-environment workflow. Application coordination belongs to the user: Loka neither stops applications nor reboots automatically, and successful live rollback reports that a reboot is required after cache preparation and verification.

Dataset rollback is sequential rather than an atomic environment-wide transition. On failure, stop, report completed, failed, and untouched members, and do not automatically retry or reverse earlier changes. See the [issue #6 interview](https://github.com/YoukouTenhouin/loka/issues/6).
