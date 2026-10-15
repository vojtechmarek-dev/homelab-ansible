# AGENTS.md

Operating contract for AI agents working in this repository. Human-facing docs are
[README.md](README.md) (layout, how to run, disaster recovery) and
[INFRASTRUCTURE.md](INFRASTRUCTURE.md) (architecture, new-service recipe, ops cheat sheet).
Read those for *what* the homelab is; this file is *how to change it safely*.

## What this repo is

Ansible for a two-site homelab: four hosts, ~20 Docker application stacks, Caddy reverse
proxy, Tailscale mesh, Borg backups. Every change is a config change — there is no build,
no test suite, no CI. Correctness is established by reading the rendered output and by the
operator applying it against real hosts.

## Hard rules

1. **Never run a playbook against a host.** Applying changes is the operator's call. Write
   the change, explain what it will do, hand them the command. This includes `--check`.
2. **Never touch `provisioning/`** unless explicitly asked. Those playbooks run `mkfs` and
   `lvg` against real disks. They are deliberately excluded from `site.yml`; do not add
   them to it, and do not "helpfully" make them idempotent.
3. **Secrets live outside this repo**, in `../ansible-secrets/secrets.yml` (Ansible Vault).
   Never inline a credential into a template, playbook, or commit. Never print decrypted
   vault contents into a transcript. `.gitignore` contains only `.vscode` — the protection
   is that secrets sit in a different directory, so a stray credential *will* be committed.
4. **Do not add a new root-level `server_*.yml` for an ordinary Docker app.** It belongs in
   `apps.yml` as a `docker_stack` play. New playbooks are for services needing bespoke
   logic (image builds, cron, sysctl, post-deploy commands).
5. **Config files the application rewrites at runtime** must not be templated without
   thought. Several apps (qbit_manage, Tdarr nodes) rewrite their own config on start; an
   Ansible-managed file then fights the app on every run.
6. **Never commit.** Do not run `git commit`, `git push`, `git rm`, or anything else that
   writes history or the index. Leave your changes in the working tree, say which files
   you touched, and let the operator review and commit. Drafting a commit message when
   asked is fine — writing the commit is not.

## Control node reality

Primary working directory is `D:\Personal\ansible` on **Windows**. Both PowerShell and Git
Bash are available. Ansible itself runs from WSL or a Linux control node, not from the
Windows shell.

Consequences for an agent working here:

- `ansible-playbook`, `ansible-lint`, `python` and `PyYAML` are generally **not available**.
  Do not plan a verification step around them.
- The Docker *CLI* is present, but the daemon is often stopped. `docker compose config`
  works without a daemon; `docker run` does not.
- `ANSIBLE_CONFIG` must point at this repo's `ansible.cfg`, because Ansible silently
  ignores a config file on a world-writable filesystem (`/mnt/d`). `.envrc` handles this
  via direnv. If an operator reports "my change did nothing", check this first.

## Layout

```
ansible.cfg          inventory = inventory.yml + ../ansible-secrets/secrets.yml; become = False
inventory.yml        hosts + functional groups (adguard_servers, docker_nodes, ...)
site.yml             orchestrator; imports playbooks in a safe order
apps.yml             14 plays, one per Docker stack, all via the docker_stack role
server_*.yml         bespoke services and host setup
experimental_*.yml   not wired into site.yml; promote or delete
provisioning/        one-time destructive disk/LVM setup — see Hard rules
roles/               base user docker ssh_security docker_stack caddy tailscale
                     borgbackup borgmatic disk_alert
templates/<service>/ Jinja2 compose + config templates, one directory per service
host_vars/           per-host facts (currently Tailscale IPs, subnet advertisement)
```

## Hosts

| host | LAN | role | domain |
|---|---|---|---|
| `kuzelovlab` | 192.168.0.20 | Proxmox VM; most apps, Caddy, Borgmatic, subnet router | `zen132.cz` |
| `medialab` | 192.168.0.30 | Proxmox VM; media stack only, 2.4 TB `/data` + SSD cache | — |
| `homelab-thinkcentre` | 192.168.66.166 | bare metal; backup target, Caddy, apps | `vojtechmarek.dev` |
| `homelab-pi` | 192.168.66.167 | Raspberry Pi; Discord bot, experiments | — |

SSH as `ansible-manager` with `~/.ssh/ansible_manager_homelab_rsa` (set in `inventory.yml`).
Tailscale IPs live in `host_vars/`, never hardcoded in templates.

## The `docker_stack` contract

Every play in `apps.yml` configures `roles/docker_stack`. Read
`roles/docker_stack/defaults/main.yml` before adding one — it documents the full variable
set. The role: makes directories → renders optional config files → renders the compose
template → `docker compose up`, restarting on change.

```yaml
- name: Deploy <Service>
  hosts: <host-or-group>
  become: true
  gather_facts: false
  tags: <service>                   # optional; add it for anything slow
  vars:
    stack_name: <service>           # required — used in task names
    stack_path: /srv/<service>      # required — project dir on the host
    stack_compose_template: <service>/docker-compose.yml.j2   # required, relative to templates/
    stack_recreate: auto            # prefer over the 'always' default: only cycle
                                    # containers whose definition actually changed
    stack_extra_dirs: []            # uniform mode/owner
    stack_dirs: []                  # per-dir {path, owner, group, mode, recurse}
    stack_config_files: []          # [{src, dest, mode}], rendered before compose
    # plus any other vars the template references
  roles:
    - docker_stack
```

Conventions that are unenforced but consistently followed:

- `gather_facts: false` — nothing in these plays needs facts, and it halves run time.
- `become: true` on every play; `ansible.cfg` sets `become = False` globally, so the
  operator supplies sudo with `-K`.
- Data paths: `/srv/<service>` for config, `/srv_fast/<service>` for databases on hosts
  with an SSD tier, `/data` on medialab for bulk media.
- Container-owned directories get `owner: "1000", group: "1000"`. LinuxServer.io images
  chown `/config` themselves; most other images do not — create those explicitly in
  `stack_dirs` rather than letting Docker create them root-owned.

## Secrets

58 distinct keys, all under a single `secrets` namespace, referenced as
`{{ secrets.<name> }}`. Naming is `<service>_<purpose>`: `mealie_db_password`,
`gotify_app_token`, `jellystat_jwt_secret`.

- Adding a key means `ansible-vault edit ../ansible-secrets/secrets.yml` — the operator
  does that, not the agent.
- Reference it in the template, then state in your summary which new vault keys the change
  requires. A missing key fails at render time with an undefined-variable error.
- Prefer graceful degradation for optional integrations: `disk_alert` and the medialab
  sync no-op when `secrets.gotify_app_token` is absent, so they can be deployed before the
  secret exists. Copy that pattern.

## Validating a change without running Ansible

This is the substitute for a test suite. Do it before handing a change over.

**Compose templates** — strip Jinja, then let the Docker CLI parse the result:

```bash
SP="$SCRATCHPAD"   # session scratchpad, never the repo
sed -e 's/{{[^}]*}}/PLACEHOLDER/g' templates/<svc>/docker-compose.yml.j2 > "$SP/check.yml"
docker compose -f "$SP/check.yml" config --quiet && echo OK
```

Catches YAML errors, bad indentation, unknown compose keys, and port or container-name
collisions. It does **not** validate Jinja logic.

**Jinja loops and conditionals** — `sed` cannot expand them, and a naive regex expands
nested loops wrongly, because a non-greedy match stops at the *inner* `{% endfor %}`.
Expand innermost loops first, or accept that only the operator's `--check --diff` run
proves it. Say which one you did.

**Variable wiring** — confirm every variable the template uses is defined in the play:

```bash
grep -oE '\{\{ *[a-z_]+' templates/<svc>/<file>.j2 | sed 's/{{ *//' | sort -u
```

Loop variables appear in `{% for %}` headers rather than `{{ }}`, so this reports them as
undefined. Read the output before believing it.

**Hand the operator** `ansible-playbook apps.yml --tags <tag> --check --diff --ask-vault-pass -K`
as the real verification step.

## Conventions

- **Comments explain *why*, not *what*.** Existing templates and playbooks carry dense
  rationale comments — a non-obvious port, a deliberate omission, a workaround for an
  upstream quirk. Match that density; it is the primary documentation in this repo.
- **Commit messages** — only ever *drafted* on request, never applied (see Hard rules):
  imperative, lowercase, present tense. A conventional-commit prefix (`docs:`, `feat:`,
  `fix:`) appears sometimes, not always — follow the surrounding history. Describe the
  change's effect, not the files touched. End with:

  ```
  Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
  ```

- **Keep changes reviewable.** The working tree frequently carries unrelated in-progress
  edits. Touch only what the task needs and name those files in your summary, so the
  operator can stage a coherent commit without untangling your work from theirs.
- **Update docs alongside the change.** A new service touches `apps.yml`, a template
  directory, the Caddyfile for its host, and usually a line in `INFRASTRUCTURE.md`.

## Adding a service

The authoritative recipe is INFRASTRUCTURE.md §3, with a worked example in §4. In short:
AdGuard DNS rewrite → Caddy site block in
`roles/caddy/templates/Caddyfile.<host>.private.j2` → template directory → play in
`apps.yml` → Cloudflare Tunnel entry if it should be public. Caddy config is per-host;
`inventory.yml` selects the template through the `caddy_template` host var.

## Known sharp edges

- **`stack_recreate` defaults to `always`**, which cycles *every* container in a stack on
  any change. For multi-service stacks (medialab runs ~12) set `auto`.
- **`community.docker` is pinned `>=3.4.0,<4.0.0`** for ansible-core 2.13 / Python 3.8
  compatibility. Do not bump it without checking the control node's ansible-core version.
- **`medialab` has no Borg repository**, by design — only its config is synced to
  kuzelovlab nightly. Anything new on medialab holding irreplaceable state must live under
  `/srv/medialab/config`, or it is not backed up at all.
- **`/data` on medialab runs near full** and the physical disk cannot be extended. Any
  change that writes there needs a space argument, not just a correctness argument.
- **No hardware video acceleration anywhere.** medialab is a Proxmox VM with emulated VGA
  (`1234:1111`, no `renderD128`), so Jellyfin and Tdarr are CPU-only. Do not add
  `/dev/dri` device mappings or GPU-flavoured encoder plugins.
