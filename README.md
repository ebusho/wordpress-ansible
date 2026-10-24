# Provision a Wordpress instance with Ansible

Single VPS stack: nginx, PHP-FPM (remi), MariaDB, Redis, Certbot,
fail2ban — fronted by Cloudflare. Targets RHEL 9-family hosts
(AlmaLinux, Rocky, CentOS Stream) with UFW as the firewall.

## Prerequisites

```shell
ansible-galaxy install -r requirements.yml
```

Roles are unpinned. Re-run with `--force` to upgrade to latest; if a
major version jumped, skim that role's changelog before running against
a live server — majors can rename variables or rework templates.

## Initial preparation

VPS hosts are usually provided with a root username and password. Before
installing anything, create a dedicated sudo user and disable root login.

1. Create `hosts` and `vars.yaml` from `hosts.example` and
   `vars.yaml.example` and fill them in. Both files are gitignored —
   never commit them.
2. Run:

```shell
ansible-playbook --user root --ask-pass prepare-host.yaml
```

## Provision the instance

```shell
ansible-playbook main.yaml
```

The playbook assumes the site is (or will be) behind Cloudflare's proxy:
ports 80/443 are opened only to Cloudflare IP ranges, and nginx restores
real visitor IPs from the `CF-Connecting-IP` header. Without a
Cloudflare-proxied domain, the firewall locks out all web traffic.

## Work on a live server

A full re-run won't wipe the site (Wordpress install steps are skipped
when it already exists), but it upgrades every package and may reboot.
Roles are tagged — run only what you need:

```shell
ansible-playbook main.yaml --list-tags
ansible-playbook main.yaml --tags system        # package upgrades
ansible-playbook main.yaml --tags php,mysql     # combine tags
```

- `configure-firewall` resets UFW and locks SSH to the public IP of the
  machine running the playbook. Run it from your usual network, not a VPN.
- MariaDB and PHP-FPM limits are tuned for ~2GB RAM; for larger
  instances raise `mysql_innodb_buffer_pool_size` and
  `php_fpm_pm_max_children`.

## Upgrading PHP

1. Set `php_version` in `vars.yaml` (e.g. `"8.5"`).
2. Run `ansible-playbook main.yaml --tags php` — enables the remi module
   stream for that version and upgrades the packages.
3. Verify on the server: `php -v`, `systemctl status php-fpm`, then load
   the site.

Snapshot the VM first — the module stream switch is the one operation
here that can go sideways. `dnf module list php` shows the current
stream.

## fail2ban

- `sshd` jail: failsafe only (SSH is already restricted to one IP by UFW).
- `wordpress` jail: watches wp-login/xmlrpc POSTs in the nginx log and
  bans offenders at Cloudflare's edge via API (zone-level IP access
  rules, bantime escalating 1d -> 4w, rules removed on expiry). A
  firewall-level ban can't work behind a proxy — packets arrive from
  Cloudflare IPs — so bans are pushed to Cloudflare instead.

Requires in `vars.yaml`: `cloudflare_api_token` (scoped token with
Zone -> Firewall Services -> Edit) and `cloudflare_zone_id`.

## Rebuilding vars.yaml for an existing server

Everything except the Wordpress admin password is recoverable from the
server: DB credentials from `wp-config.php` (`DB_NAME`, `DB_USER`,
`DB_PASSWORD`), MySQL root password from `/root/.my.cnf`, domain from
the nginx vhost. `mysql_root_password` and the `mysql_users` password
must match reality — the mysql role authenticates with them.
