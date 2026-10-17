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

## Wiring opencode up

`shared_secret` becomes `BRIDGE_API_KEY` everywhere: what opencode presents to
the router, what the router presents to each bridge, and each bridge's own key.
One string, all three jobs.

opencode runs a shared background server, so exporting the variable in a
terminal will not reach it. Use the service's managed environment:

```sh
opencode service set env BRIDGE_API_KEY <shared_secret>
```

Then add to `~/.config/opencode/opencode.jsonc`, then restart the service by
running any opencode command. It appears in `opencode auth list` with type
`environment`, which is correct — it is not a saved account, so `auth logout`
will not remove it.

```jsonc
"providers": {
  "cc": {
    "name": "Command Code (pooled)",
    "env": ["BRIDGE_API_KEY"],
    "package": "@opencode/ai/providers/openai-compatible",
    "settings": {
      "baseURL": "http://<arm-tailnet-ip>:9992/v1",
      "encoding": "none"
    },
    "models": { "gpt-5.3-codex": { "name": "GPT-5.3 Codex" } }
  }
}
```

## Getting the keys

`~/.commandcode/auth.json` holds one key at a time, so capture after each login
before doing the next. `/connect` → Command Code in opencode, then:

```sh
python3 -c "import json,os;k=json.load(open(os.path.expanduser('~/.commandcode/auth.json')))['command-code']['key'];print(k)"
```

## When something breaks

**A 403 naming a version.** Upstream raised its floor. Re-run the playbook; the
pin task picks up the current published version.

**A new bridge release, or models you want.** Change `cc_bridge_tag` to the
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
