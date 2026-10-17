# Command Code pooled gateway

Two bridge containers, one Go account each, on two Oracle VPSes.
`commandcode-router` pools them from the third. opencode runs on the laptop and
talks to the router over the tailnet.

Ansible owns the fleet: the source tree, the image build, the env file, and the
CLI version pin. No cron jobs, no GitHub workflows, no scripts on the VPSes, and
no registry.

## Steps

- [ ] Log into each account in turn and capture its key before the next login
- [ ] Lay out the playbook, inventory and vault files
- [ ] Run the playbook once; it clones, builds, writes env files and starts up
- [ ] Check the router's `/v1/models` against what you actually want to use
- [ ] Add a `cc` provider to opencode.jsonc pointing at the router
- [ ] Smoke test a real agent turn, then prove failover by stopping one bridge

## Layout

```
opencode (laptop)
   └──tailnet──→ router :9992          ARM 24GB / 4 OCPU  · group: cc_router
                   ├──→ VPS-1 bridge   Oracle AMD 1GB    · group: cc_bridge
                   └──→ VPS-2 bridge   Oracle AMD 1GB    · group: cc_bridge
```

The router sits on the big box because it is the single point every request
passes through. The 1GB boxes only stream bytes and keep per-key bookkeeping,
which is nothing next to Node's own footprint. Measured on this machine:
`node:24-alpine` is 166 MB on disk and idle Node 24 RSS is 50 MB, so the whole
stack is about 300 MB including the daemon.

Two accounts means two bridges. The third box is the router, not a third bridge.
If a third account arrives, the ARM box has the headroom to host it too.

Each account egresses from its own VPS IP, so the two never share a source
address. That is the entire reason for the split. Throughput is identical either
way, because quota is per account.

## Repo layout

```
command_code_gateway/
├── PLAN.md
├── inventory.yml
├── site.yml
├── vault.yml                  # encrypted: shared secret + both keys
├── group_vars/
│   └── all.yml                # repo url, tag, bridge port
└── files/
    ├── bridge.env.j2
    └── router.env.j2
```

`inventory.yml`, two groups so one playbook drives both roles:

```yaml
all:
  children:
    cc_bridge:
      hosts:
        oracle1:
          ansible_host: <oracle1-tailnet-ip>
        oracle2:
          ansible_host: <oracle2-tailnet-ip>
    cc_router:
      hosts:
        armbox:
          ansible_host: <arm-tailnet-ip>
```

`vault.yml` holds everything secret and nothing else. One dict, so adding a
third account is a dict entry rather than another group-vars file:

```yaml
vault_data:
  shared_secret: "<random>"
  cc_keys:
    oracle1: "user_..."
    oracle2: "user_..."
```

Create it with `ansible-vault create vault.yml`, then edit. Every task that
touches it sets `no_log: true`.

## What the playbook does

One play, three phases.

**Source.** Clone or update the author's repo into
`/opt/commandcode-bridge` on each box, checked out at a tag you choose. Building
from a tag rather than `main` makes the result reproducible and gives rollback
for free.

```yaml
- name: commandcode-bridge source
  ansible.builtin.git:
    repo: https://github.com/yelixir-dev/commandcode-bridge
    dest: /opt/commandcode-bridge
    version: "{{ cc_bridge_tag }}"   # a release tag, not main
```

**Env file.** Render from the vault, mode 0600, `no_log: true`. The template
picks the key for the host it is running on, which is the only per-host
difference:

```jinja
HOST=0.0.0.0
BRIDGE_API_KEY={{ vault_data.shared_secret }}
COMMANDCODE_API_KEY={{ vault_data.cc_keys[inventory_hostname] }}
COMMANDCODE_CLI_VERSION={{ cc_cli_version }}
```

**Compose.** `build: always` so a version bump to the tag is picked up, and
`state: present` so the whole run is idempotent.

```yaml
- name: up
  community.docker.docker_compose_v2:
    source: /opt/commandcode-bridge
    build: always
    state: present
    recreate: always
```

### Long builds and SSH timeouts

`docker_compose_v2` is synchronous, so a 20-minute build on a 1GB box holds the
task open and can outlive your SSH timeout. If that happens, use
`ansible.builtin.command` with `async` instead:

```yaml
- name: up (long build)
  ansible.builtin.command: docker compose up -d --build
  args:
    chdir: /opt/commandcode-bridge
  async: 3600
  poll: 15
  changed_when: true
```

## The version pin, as a task

This is the part that replaces cron. Rather than scheduling anything, the
playbook reads the published version and writes it into every env file, so the
pin is refreshed whenever you choose to run it — no timer, no silent 6am
restart.

```yaml
- name: current Command Code CLI version
  ansible.builtin.command: npm view command-code version
  register: cc_version_probe
  changed_when: false
  delegate_to: localhost
  check_mode: false

- name: pin
  ansible.builtin.set_fact:
    cc_cli_version: "{{ cc_version_probe.stdout }}"
```

`delegate_to: localhost` because the Oracle hosts have no npm, only the
container does. Your Mac already has node.

The bridge hardcodes CLI `1.74.0` in `src/config.ts`, and the published CLI is
**1.83.0**, so the baked default is nine versions stale and will eventually
hard-403. `COMMANDCODE_CLI_VERSION` in the env file overrides it at runtime, so
the pin never requires a rebuild.

## Compose

Docker with compose on all three boxes. `network_mode: host` rather than
published ports: it sidesteps rootless port publishing entirely, and
`HOST=0.0.0.0` does the binding. On the ARM box you could publish normally, but
keeping the two files identical is worth more than the tidiness.

Both bridges share `compose.yaml` from the cloned repo, run as a bridge:

```yaml
services:
  bridge:
    build: .
    container_name: cc-bridge
    restart: unless-stopped
    network_mode: host
    env_file: ./bridge.env
```

The router needs a different entrypoint, so it gets its own file next to it:

```yaml
services:
  router:
    build: .
    container_name: cc-router
    restart: unless-stopped
    network_mode: host
    env_file: ./router.env
    command: ["node", "dist/router-index.js"]
```

Router env file. `COMMANDCODE_ROUTER_BACKENDS` takes comma-separated `id=url`
pairs. One `BRIDGE_API_KEY` does double duty: what the router demands from
clients, and what it sends to each backend.

```
HOST=0.0.0.0
BRIDGE_API_KEY={{ vault_data.shared_secret }}
COMMANDCODE_ROUTER_BACKENDS=vps1=http://<oracle1-tailnet-ip>:9992,vps2=http://<oracle2-tailnet-ip>:9992
```

## Capture the keys

There is currently one key, mirrored by the plugin across three files. The
`command-code` slot in `~/.commandcode/auth.json` holds a single token, so each
login replaces the last one. Capture after every login, never batch.

```sh
mkdir -p ~/.commandcode/pool && chmod 700 ~/.commandcode/pool

# then, once per account, after /connect -> Command Code completes:
python3 -c "import json,os;k=json.load(open(os.path.expanduser('~/.commandcode/auth.json')))['command-code']['key'];print(k)" \
  > ~/.commandcode/pool/key-1.txt
chmod 600 ~/.commandcode/pool/key-1.txt
```

Confirm both without printing either, then paste into `vault.yml`:

```sh
python3 -c "
import hashlib,glob
for f in sorted(glob.glob('/Users/rishav/.commandcode/pool/key-*.txt')):
    print(hashlib.sha256(open(f).read().strip().encode()).hexdigest()[:12], f)"
```

Two matching hashes means a login overwrote a capture.

One thing to watch: opencode v2 stores provider credentials in SQLite, not in
`auth.json`. `~/.commandcode/auth.json` is the Command Code CLI's own file and
the plugin mirrors to it, but confirm the hash changed after each `/connect`
rather than assuming the mirror still fires. The comparison above is the check.

## opencode config

`vault_data.shared_secret` becomes `BRIDGE_API_KEY`, and it does triple duty:

1. the credential opencode presents to the router
2. the credential the router presents to each bridge
3. each bridge's own `BRIDGE_API_KEY`

One string, all three. Hand it to opencode through the **server's** managed
environment, not your shell — opencode runs a shared background service, and
exporting in a terminal does not reach it:

```sh
opencode service set env BRIDGE_API_KEY <shared_secret>
opencode service unset env BRIDGE_API_KEY     # to remove
```

That persists in the service config and survives reboots and relaunches from a
GUI client. It stops the running service once; the next opencode command brings
it back with the value. It shows up in `opencode auth list` with type
`environment`, which is expected — it is not a saved account, so `auth logout`
cannot remove it.

Then add the provider beside the existing `plugins` and `mcp` in
`~/.config/opencode/opencode.jsonc`. Take model IDs from the router's
`/v1/models` rather than from this file:

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
    "models": {
      "gpt-5.3-codex": { "name": "GPT-5.3 Codex" }
    }
  }
}
```

`env` is required because the router is off loopback. `encoding: "none"`
because gateways reject compressed bodies.

## Maintenance

Two paths, neither scheduled. Both start with `ansible-playbook site.yml`.

**A 403 names a version.** Upstream raised its floor. Edit `cc_bridge_tag` or
let the probe task refresh the pin, re-run the playbook. The compose step
recreates the containers. No rebuild unless you also moved the tag.

**A new bridge release, a bug, or models you want.** Set
`cc_bridge_tag` to the newer tag, re-run. The clone updates, the image
rebuilds, containers recreate. Rollback is setting the tag back.

The playbook is idempotent, so re-running it with nothing changed is safe and a
no-op except for the image rebuild.

## Tradeoffs

**What it costs:** failover sits one layer up. With one key per bridge there is
nothing for a bridge to fail over to, so it all lands on the router — and the
router cools a failed backend for a flat 60 seconds, ignoring `Retry-After`. An
account whose 5-hour window is spent gets re-probed every minute instead of
parked until its real reset time. Nothing breaks; the other account still serves
everything. It just chatters. A single container with both keys parks it
properly, at the cost of putting both accounts on one IP.

**What survives the split:** the bridge sends `x-project-slug` derived from the
working directory, so both accounts working the same repo present the same
slug. Separate IPs, shared slug. Whether Command Code cares is undocumented and
unverified.

## Risks

- **Building on 1GB.** The Dockerfile's build stage runs `npm run verify`, the
  full typecheck, lint and test suite. On one vCPU with an 8GB swapfile this is
  slow and thrashy, and might fail on a test timeout rather than complete. Your
  first playbook run is the experiment. If the Oracle builds prove unreliable,
  build on the ARM box and add `docker save` / `docker load` tasks instead —
  every host already has the image pull as a fallback. Do not patch the
  Dockerfile to skip the tests.
- **Version gate.** The symptom is a 403 naming a version, which does not look
  like a rate limit and will send you looking in the wrong place.
- **Vault password.** Losing it means the keys are gone and every account needs
  a fresh browser login. Keep it in a password manager, not only in your head.
- **Tailnet exposure.** All three boxes bind `0.0.0.0`, so `BRIDGE_API_KEY` is
  the only control. Use a real secret and confirm tailnet ACLs.
- **Frozen catalog.** `refreshProviderModelCatalog` in `src/server.ts` fetches
  `GET /provider/v1/models` at startup with no `Authorization` header. That is
  the Provider API, which 403s for Go plans, so the fetch fails and the static
  table is kept. Go accounts are permanently on the bundled **1.74.0 catalog of
  85 models**. Your plugin's catalog is 87 at 1.79.2, so the bridge serves
  fewer models than the plugin it replaces. Only a newer bridge release fixes
  this.
- **Not verified:** whether Command Code correlates accounts by source IP,
  whether it rejects a version too *new* as well as too old, whether a second
  login invalidates the first account's session, and whether
  `docker compose` on Oracle Linux handles `network_mode: host` as expected.
- **Terms.** Two $1 accounts to double throughput is a question for Command
  Code. The bridge README states it does not bypass account limits and that
  terms still apply.
