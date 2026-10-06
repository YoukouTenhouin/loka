# Use bootfs in one configured boot pool

Loka may manage environments across multiple pools, but the machine's persistent default comes from `bootfs` in one explicitly configured boot pool. The supported ZFSBootMenu configuration selects that pool without a dataset-valued `zbm.prefer` override, because such an override would make changing `bootfs` ineffective. This installation constraint trades arbitrary bootloader preference configurations for a default that loka can identify and change consistently.
