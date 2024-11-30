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
