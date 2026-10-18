# command_code_gateway

Ansible that runs a pooled [Command Code](https://commandcode.ai) gateway across
three VPSes on a tailnet. Two $1 Go accounts, one bridge each so neither shares a
source IP, pooled by the bridge's own `commandcode-router`. opencode runs on the
laptop and talks to the router.

Full design and rationale in [PLAN.md](PLAN.md).

## What it does

Per host the playbook:

1. installs Docker CE, compose and buildx (Oracle Linux, Ubuntu/Debian and
   RHEL-family all handled)
2. syncs the upstream bridge source at a pinned release tag
3. renders an env file from Ansible Vault, mode 0600
4. builds the image and starts the container under compose
5. asserts the container is actually running

Plus one nicety: it reads the published Command Code CLI version from npm and
pins it, so the bridge's stale baked-in version never hard-403s. It also warns
when a newer bridge release exists, without following it.

## Run it

### 1. Fill in the inventory

```sh
$EDITOR inventory.yml
```

Replace the three `REPLACE_WITH_*_TAILNET_IP` values and the two
`REPLACE_WITH_SSH_USER` ones. The hostnames `oracle1`, `oracle2` and `armbox`
matter — they key the vault and the per-host project slug.

### 2. Get the two Command Code keys

`~/.commandcode/auth.json` holds **one** key at a time, so each login replaces
the last. Capture after each one, or you lose it.

```sh
mkdir -p ~/.commandcode/pool && chmod 700 ~/.commandcode/pool
```

Then, in opencode, `/connect` → Command Code. After it completes:

```sh
python3 -c "import json,os;k=json.load(open(os.path.expanduser('~/.commandcode/auth.json')))['command-code']['key'];print(k)" \
  > ~/.commandcode/pool/key-1.txt
chmod 600 ~/.commandcode/pool/key-1.txt
```

Repeat with the second account, writing `key-2.txt`. Confirm you got both
without printing either:

```sh
python3 -c "
import hashlib,glob
for f in sorted(glob.glob('/Users/rishav/.commandcode/pool/key-*.txt')):
    print(hashlib.sha256(open(f).read().strip().encode()).hexdigest()[:12], f)"
```

Two different hashes. Two matching hashes means a login overwrote a capture.

One caveat: opencode v2 stores credentials in SQLite, and `auth.json` is the
Command Code CLI's own file. If the hash does not change after a login, stop and
check — otherwise you will reuse account 1's key on account 2, which is the one
failure here that produces no error message.

### 3. Create and encrypt the vault

```sh
cp vault.yml.example vault.yml
openssl rand -hex 32        # copy this into shared_secret
$EDITOR vault.yml           # shared_secret, then both keys from step 2
ansible-vault encrypt vault.yml
```

Pick a vault password and put it in your password manager. Losing it means
re-doing step 2 from scratch.

### 4. Dry run against one host

```sh
ansible all -m ping
ansible-playbook site.yml --limit oracle1 --check --ask-become-pass
```

The version pin lives inside the fleet play specifically so `--limit` does not
skip it.

### 5. Deploy

```sh
ansible-playbook site.yml --ask-become-pass
```

First run builds an image on each box, including the upstream project's full
verify suite. On the two 1GB Oracle boxes that is slow, and it is the one part
this setup has never tested. Expect several minutes, not seconds.

### 6. Wire opencode up

Write the same `shared_secret` to a local key file:

```sh
umask 077
printf '%s' '<shared_secret>' > ~/.config/opencode/cc-bridge.key
```

Add to `~/.config/opencode/opencode.jsonc`, then restart opencode:

```jsonc
"providers": {
  "cc": {
    "name": "Command Code (pooled)",
    "package": "@opencode/ai/providers/openai-compatible",
    "settings": {
      "baseURL": "http://<armbox-tailnet-ip>:9992/v1",
      "apiKey": "{file:~/.config/opencode/cc-bridge.key}",
      "encoding": "none"
    },
    "models": {
      "deepseek/deepseek-v4.1-flash": { "name": "DeepSeek V4.1 Flash" }
    }
  }
}
```

`~/.config/opencode/` is a plain local directory whose config files are
symlinks into your dotfiles repo, so that key file is never tracked.

**The `models` block is required.** opencode does not call a custom provider's
`/v1/models` — discovery is special-cased for Ollama, LM Studio and vLLM only.
Without it you get `Model unavailable: cc/<anything>`.

### 7. Prove it works

```sh
curl -sS http://<armbox-tailnet-ip>:9992/health
curl -sS -H "Authorization: Bearer $(cat ~/.config/opencode/cc-bridge.key)" \
  http://<armbox-tailnet-ip>:9992/v1/models | head -c 400
```

Then a real turn in opencode on `cc/deepseek/deepseek-v4.1-flash`, and stop one
bridge (`ansible oracle1 -a 'docker compose -f /opt/cc/bridge/compose.yml stop'`)
to confirm the other still serves.

## Maintenance

Two paths, neither scheduled. The version pin refreshes itself on every run.

**A 403 names a version.** Just re-run the playbook; the pin picks up the
current published version. No rebuild.

**A new bridge release, or models you want.** Change `cc_bridge_tag` in
`group_vars/all.yml` and re-run. Rollback is changing it back.

## Gotchas that cost real time

- **Only one model is allowlisted.** The bridge enables 6 of its 85 bundled
  models and rejects the rest with a 400. `cc_allowed_models` in
  `group_vars/all.yml` extends that set. A second model must be added to both
  that list and the opencode `models` block, or one side 400s.
- **Both keys on one box would share one IP.** The whole point of the split is
  a distinct egress IP per account. Collapsing to a single container is cheaper
  but puts both accounts behind one address.
- **The router ignores `Retry-After`.** It cools a failed backend for a flat
  60s, so an account whose window is spent gets re-probed each minute rather
  than parked until it resets. Harmless with two accounts, noisy.
- **`x-project-slug` is per-host** (`working_dir` in the compose template), so
  the accounts do not share a slug. Only the basename is sent.

## Secrets

`vault.yml` is gitignored. Only `vault.yml.example` is committed.
`~/.config/opencode/cc-bridge.key` lives outside any repo.

## Requirements

- Docker installs itself; nothing else is assumed beyond SSH and sudo
- Ansible on the controller, plus `git` and `npm` (the version probe uses the
  published npm package)
