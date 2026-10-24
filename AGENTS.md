# AGENTS.md

Ansible playbook provisioning a single Cloudflare-fronted Wordpress VPS
(RHEL 9 family: AlmaLinux/Rocky/CentOS Stream, dnf-based). No tests, no
CI — `ansible-playbook main.yaml --syntax-check` is the only gate.

## Layout

- `main.yaml` — everything; roles are tagged, run subsets with `--tags`.
- `prepare-host.yaml` — first-run bootstrap as root (sudo user, sshd).
- `roles/*` — local roles; `geerlingguy.*` come from Galaxy
  (`requirements.yml`, deliberately unpinned — refresh with `--force`
  and skim changelogs of major bumps).
- `vars.yaml` + `hosts` — gitignored secrets, never commit. Recreate
  from `vars.yaml.example` / `hosts.example`.

## Non-obvious design decisions

- The site is assumed to sit behind Cloudflare's proxy: UFW opens 80/443
  to Cloudflare IP ranges only, nginx restores real visitor IPs from
  `CF-Connecting-IP`. Without CF the firewall blocks all web traffic.
- fail2ban's wordpress jail bans at Cloudflare's edge via API v4
  (zone-scoped token), not iptables: behind a proxy the VPS only sees
  Cloudflare edge IPs. Don't "fix" this back to a firewall action.
  SELinux on EL9 also forbids fail2ban from writing nginx config or
  reloading nginx, which rules out nginx-deny-style actions.
- `geerlingguy.php` needs `php_packages_state: latest` to actually
  upgrade versions; the `module_stream_switch` pre-task in main.yaml is
  required for `dnf module enable php:remi-X` to switch streams on EL9.
- MySQL password updates are disabled (`*_password_update: false`) so
  re-runs can't rotate credentials under a live site; the vars must
  still match reality because the role authenticates with them.
- MariaDB/FPM settings are sized for ~2GB RAM.

## Conventions

- Conventional commits chosen by effect; one commit per logical change;
  never push unless explicitly asked.
- Verify role behavior against the *installed* role versions
  (`ansible-galaxy role list`), not upstream latest.
