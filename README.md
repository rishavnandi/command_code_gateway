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

Write it to a local file and read it with a `{file:...}` substitution — the same
pattern this config already uses for Context7. `~/.config/opencode/` is a plain
local directory whose config files are symlinks into the dotfiles repo, so a
real file there is never tracked.

```sh
chmod 600 ~/.config/opencode/cc-bridge.key
```

Then add to `~/.config/opencode/opencode.jsonc` (a symlink to your dotfiles
repo) and restart opencode:

```jsonc
"providers": {
  "cc": {
    "name": "Command Code (pooled)",
    "package": "@opencode/ai/providers/openai-compatible",
    "settings": {
      "baseURL": "http://<arm-tailnet-ip>:9992/v1",
      "apiKey": "{file:~/.config/opencode/cc-bridge.key}",
      "encoding": "none"
    },
    "models": { "deepseek/deepseek-v4.1-flash": { "name": "DeepSeek V4.1 Flash" } }
  }
}
```

A missing key file substitutes an empty string, so opencode sends an empty
Bearer and the router answers 401. If the provider authenticates as nobody,
check the file exists before suspecting the tailnet.

**The `models` block is required** — opencode will not call your router's
`/v1/models`. Discovery is special-cased for Ollama, LM Studio and vLLM only.
For any other OpenAI-compatible endpoint the docs say to list models
explicitly. Without them the provider exists but nothing is selectable, and
you get `Model unavailable: cc/<anything>`. List the handful you use, not all
85 the bridge serves.

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

- Only **one** model is allowlisted (`cc_allowed_models` in
  `group_vars/all.yml`), because that is all this setup needs. The bridge
  enables 6 of its 85 bundled models by default and rejects everything else
  with a 400, so any second model must be added to both that list and the
  opencode `models` block.
- The router cools a failed backend for a flat 60s and ignores `Retry-After`, so
  an exhausted account is re-probed each minute rather than parked until its
  window resets. Harmless with two accounts; noisy.
- `x-project-slug` now differs per host (`working_dir` in the compose
  template), so the two accounts do not share a slug.
