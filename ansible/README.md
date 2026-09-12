# Ansible

Configuration management playbooks for FreeIPA, Authentik, Wazuh agents, PatchMon agents, and general host hardening.

## Usage
```bash
ansible-playbook -i inventory/hosts site.yml
```

`inventory/hosts` (real inventory) is gitignored — copy from `inventory/hosts.example`.

## TODO
- [ ] `site.yml` entrypoint
- [ ] Role: freeipa-client (domain join)
- [ ] Role: wazuh-agent
- [ ] Role: patchmon-agent
- [ ] Role: base-hardening
