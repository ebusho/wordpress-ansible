# Provision Wordpress instance with Ansible

## Prerequesites

```shell
ansible-galaxy install -r requirements.yml
```

## Initial preparation

VPS hosts are usually provided with root username and password. Before installing anything, create a dedicated sudo user and disable root login.

1. Create `hosts` and `vars.yaml` files from `hosts.example` and `vars.yaml.example` files. and fill in variables appropriately.

2. Run playbook:
```shell
ansible-playbook --user root --ask-pass prepare-host.yaml
```

## Provision instance with Wordpress

```shell
ansible-playbook main.yaml
```

## Work on a live server

A full re-run won't wipe the site (WordPress install steps are skipped when
the site already exists), but it upgrades every package and may reboot.
Roles are tagged so you can run only what you need:

```shell
ansible-playbook main.yaml --list-tags
ansible-playbook main.yaml --tags system        # package upgrades
ansible-playbook main.yaml --tags php,mysql     # combine tags
```

## Upgrading PHP

1. Set `php_version` in `vars.yaml` (e.g. `"8.3"`).
2. Run `ansible-playbook main.yaml --tags php` — this switches the remi
   module stream and upgrades the packages.
3. Verify on the server: `php -v`, `systemctl status php-fpm`, then load
   the site.

Snapshot the VM first — the module stream switch is the one operation
here that can go sideways. `dnf module list php` shows the current
stream.
