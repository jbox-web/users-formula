# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `salt-formulas/CLAUDE.md` describes the fork collection, the shared
toolchain (Kitchen, pre-commit, CI) and the `template-formula` layout. This
file only records where `users-formula` departs from that layout.

## Commands

```bash
bundle install
bin/kitchen list                                # 6 instances: {default,vimrc} x 3 platforms
bin/kitchen test default-debian-13-3008-2
bin/kitchen verify vimrc-debian-11-3006-27
pre-commit run --all-files
```

The `vimrc` suite pulls `vim-formula` from `github.com/saltstack-formulas`
(upstream, not a jbox-web fork) as a Kitchen dependency.

## Architecture — not the template layout

This formula predates the `template-formula` structure. None of the following
holds here:

- no `package/` / `config/` / `service/` split — one flat `.sls` per feature;
- no `osarchmap` / `osfamilymap` / `osmap` / `osfingermap` YAML files — the
  per-`os_family` values (`sudoers_dir`, `root_group`, `shell`,
  `googleauth_package`, `polkit_dir`, …) are inlined in `users/map.jinja`;
- no `libtofs.jinja` — templates are picked by a plain two-entry `source:`
  list, `salt://users/files/<feature>/<username>/<file>` then the shared
  default (`bashrc.sls`, `profile.sls`, `vimrc.sls`);
- `tplroot` is hardcoded as `users` everywhere except `_mapdata/init.sls`;
- no `_mapdata` reference dumps under `test/integration/*/files/`: the
  `users._mapdata` state runs in both suites but nothing asserts on it.

### Two pillar namespaces

The `users:` pillar key is the user dictionary itself, so the formula's own
settings cannot live there. They live under `users-formula:` instead:

- `users-formula:lookup` — overrides the `map.jinja` dict (merged on top of
  `defaults.yaml:users` and the `os_family` block);
- `users-formula:use_vim_formula` — gates `users.vimrc` entirely.

`groups:` and the legacy `absent_users:` list are two more top-level pillar
keys read by `init.sls`. `docs/README.rst` and `pillar.example` are the
reference for per-user keys.

### `init.sls` is the formula

`users/init.sls` (≈600 lines) does almost everything, in one loop over
`pillar['users']`: groups, home directory, `user.present`, SSH keys /
`authorized_keys` / `ssh_config` / `known_hosts`, the per-user sudoers file
(written then validated with `visudo -cf`, deleted on failure), Google
Authenticator secrets, gitconfig (only if `git` is on the minion), then
`user.absent` for users flagged `absent`.

A first pass over the same pillar decides which side states to `include:` —
`users.sudo` (any `sudouser`/`sudoonly`), `users.googleauth` (any
`google_auth`), `users.user_files` (any `user_files.enabled`),
`users.polkit` (any `polkitadmin`). Those side states install packages and
global files; the per-user resources stay in `init.sls`. Conversely,
`bashrc`, `profile`, `vimrc` and `user_files` each `include: users`, and are
targeted on their own.

Every per-user state ID embeds the username (`users_<name>_user`,
`users_<name>_<group>_group`, …); requisites across files rely on those
exact IDs, so renaming one breaks the side states.

### Pitfalls already met in tests

- `user.present` requires `- group: <prime group>`; with `uid` set and no
  `prime_group.gid`, the primary group gets `gid = uid`.
- The test pillar keeps `buser` at `uid: 2001`, outside the range `useradd`
  allocates, because a shared uid with an auto-allocated user makes home
  ownership read back as the wrong user.
- No `expire` in the test pillar: on Salt 3008 an expired account cannot be
  impersonated (`su`) by the states that manage its home.
- The SSH key file states (`ssh_keys`, `ssh_keys_pillar`) set
  `show_changes: False` so private keys from pillar never land in the state
  output; the Google Authenticator secrets do not.
