# command_code_gateway

Ansible that runs a pooled [Command Code](https://commandcode.ai) gateway across
three VPSes on a tailnet. Two $1 Go accounts, one bridge each, so neither shares
a source IP, pooled by the bridge's own `commandcode-router`.

Full design and rationale in [PLAN.md](PLAN.md).

## What it does

Per host: sync upstream source at a pinned tag, render an env file from Ansible
Vault, build the image, start the container. Plus one nicety — it reads the
published Command Code CLI version from npm and pins it, which is what keeps the
bridge's stale baked-in version from hard-403ing.

## Setup

```sh
cp vault.yml.example vault.yml
$EDITOR vault.yml                    # shared_secret + one key per bridge host
ansible-vault encrypt vault.yml
$EDITOR inventory.yml                # tailnet IPs
$EDITOR group_vars/all.yml           # a release tag from:
git ls-remote --tags https://github.com/yelixir-dev/commandcode-bridge
```

Then run it:

```sh
ansible-playbook site.yml
```

The hosts need Docker installed and a sudo-capable login; the playbook does not
install Docker.

## Getting the keys

`~/.commandcode/auth.json` holds one key at a time, so capture after each login
before doing the next. `/connect` → Command Code in opencode, then:

```sh
python3 -c "import json,os;k=json.load(open(os.path.expanduser('~/.commandcode/auth.json')))['command-code']['key'];print(k)"
```

## When something breaks

**A 403 naming a version.** Upstream raised its floor. Re-run the playbook; the
pin task picks up the current published version.

**A new bridge release, or models you want.** Change `cc_bridge_version` to the
newer tag and re-run. Rollback is changing it back.

## Secrets

`vault.yml` is gitignored. Only `vault.yml.example` is committed. The vault
password is load-bearing for both accounts — store it in a password manager.

## Known limits

- Go-plan accounts cannot use the bridge's live model catalog; they are pinned to
  the version bundled in the release. Only a newer bridge release changes it.
- The router cools a failed backend for a flat 60s and ignores `Retry-After`, so
  an exhausted account is re-probed each minute rather than parked until its
  window resets. Harmless with two accounts; noisy.
- Both accounts send the same `x-project-slug` for the same repo. Separate IPs,
  shared slug.
