---
tags:
  - Ansible
---
# Using Fully Qualified Collection Names (FQCN) like `ansible.builtin.dnf`, `ansible.builtin.systemd`
- **Ansible 2.10+ best practice** - Avoids namespace collisions when multiple collections provide modules with same names
    
- **Explicit dependencies** - Makes it clear which collection provides each module
    
- **ansible-lint compliance** - Modern linting tools enforce FQCN usage (you have `# noqa` comments showing you ran ansible-lint)
    
- **Future compatibility** - Required for Ansible 3.0+