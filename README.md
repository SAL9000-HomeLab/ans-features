# ans-features

Adds and removes Windows features and capabilities with one role, `ans_features`. Each feature has its own variable,
set on the inventory, group or host. It is meant to be included from [ans-defos] as part of the default OS
configuration, and [site.yml](site.yml) runs it on its own.

Only the features whose variable is set are touched. The role changes nothing on hosts that are not Windows.

| Feature | Variable | Tag | What Windows calls it |
| --- | --- | --- | --- |
| OpenSSH Server | `ans_features_openssh_server` | `openssh` | Capability `OpenSSH.Server~~~~0.0.1.0` |
| Telnet Client | `ans_features_telnet_client` | `telnet` | Optional feature `TelnetClient` |

## Requirements

| Requirement | Version |
| --- | --- |
| ansible-core | >= 2.16 |
| Python | >= 3.10 |

- Targets: Windows Server 2025 and Windows 11, reached over PSRP as an administrator. The controller needs the Python
  packages in [requirements.txt](requirements.txt).
- Collections: `ansible.windows` 3.6.0 or later (for `win_capability`), from [requirements.yml](requirements.yml).

The role gathers the `min` fact subset itself when facts were not gathered, so it works under plays with
`gather_facts: false`.

## Variables

Defaults are in [roles/ans_features/defaults/main.yml](roles/ans_features/defaults/main.yml).

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_features_openssh_server` | `""` | `present`, `absent`, or empty to leave OpenSSH Server alone. |
| `ans_features_telnet_client` | `""` | `present`, `absent`, or empty to leave Telnet Client alone. |
| `ans_features_reboot` | `false` | Restart the host when a change needs it. Otherwise the role reports it. |
| `ans_features_reboot_timeout` | `1800` | Seconds `win_reboot` waits for the host to come back. |

Because each feature has its own variable, a host can override one feature without repeating the rest. Example, in
group vars or the AWX inventory:

```yaml
# windows group
ans_features_openssh_server: present
ans_features_telnet_client: present

# one host that must not run sshd
ans_features_openssh_server: absent
```

## Behavior

- **OpenSSH Server, present:** adds the capability if it is missing, sets `sshd` to start automatically and starts
  it. Server 2025 ships with the capability installed and `sshd` disabled, so there only the service changes. On
  Windows 11 the capability is downloaded from Windows Update. A host that gets its updates from WSUS needs the Group
  Policy setting "Specify settings for optional component installation and component repair" to allow Windows Update
  or name a Features on Demand source, or the add fails. The first start of `sshd` creates
  `C:\ProgramData\ssh\sshd_config`, which [ans-local-accounts] edits.
- **OpenSSH Server, absent:** stops and disables `sshd`, then removes the capability.
- **Firewall:** adding the capability creates the built-in rule `OpenSSH-Server-In-TCP`, open to any address.
  [ans-firewall] disables that rule and adds its own, limited to the management addresses, so run it after this
  role. [ans-defos] does.
- **Telnet Client:** turns the optional feature on or off. Its files ship with Windows, so nothing is downloaded.
- **Restarts:** neither feature normally needs one. When a change does, the role restarts the host if
  `ans_features_reboot` is true, and otherwise prints a message at the end.

## Adding a feature

1. Add `ans_features_<name>: ""` to `defaults/main.yml` with a comment.
2. Add `tasks/<name>.yml`, which changes the feature and adds its `reboot_required` result to
   `ans_features__reboot_required`.
3. In `tasks/main.yml`, add the variable to the check list and an `include_tasks` with its own tag.
4. Add a row to the tables here and in [ans-defos], and a changelog line.

## Inventory

[site.yml](site.yml) runs the role on the `windows` group over PSRP, as in the placeholder
[inventory/hosts.yml](inventory/hosts.yml). Put real hosts in the AWX inventory, or in a gitignored `local/` copy.

## Development and CI

The workflows and lint configs are the shared ones from `ans-template`:

- **Ansible CI** (`.github/workflows/ansible-ci.yml`), on every pull request: `yamllint`,
  `ansible-playbook --syntax-check site.yml`, `ansible-lint`.
- **Linting Validation** (`.github/workflows/ci.yml`), on pull requests: markdownlint, linkspector, yamllint.

Run the same checks locally:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install "ansible-core>=2.16" "ansible-lint>=6" "yamllint>=1.30" -r requirements.txt
ansible-galaxy collection install -r requirements.yml
yamllint -f parsable .
ansible-playbook -i inventory/hosts.yml --syntax-check site.yml
ansible-lint .
npx markdownlint-cli2 "**/*.md" "#.venv"
```

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change. Pushing a `vX.Y.Z` tag
publishes that version's section as a GitHub release.

## License

MIT, see [LICENSE](LICENSE).

[ans-defos]: https://github.com/SAL9000-HomeLab/ans-defos
[ans-firewall]: https://github.com/SAL9000-HomeLab/ans-firewall
[ans-local-accounts]: https://github.com/SAL9000-HomeLab/ans-local-accounts
