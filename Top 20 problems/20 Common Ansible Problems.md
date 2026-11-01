---
tags:
  - Ansible
  - Troubleshooting
---

Knowing these problems helps you debug automation without making uncontrolled changes.

---

## Main idea

Most Ansible problems involve inventory, connectivity, privilege escalation, Python, variables, YAML, or module behavior.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. No hosts matched

- **Why it happens:** The host pattern does not match the loaded inventory.
- **Quick sign:** The play is skipped with no hosts matched.
- **Solution:** Verify inventory source, group names, and limit patterns.
- **Useful commands:** `ansible-inventory -i inventory --graph`, `ansible-playbook -i inventory site.yml --list-hosts`

---

## 2. Wrong configuration file is loaded

- **Why it happens:** Configuration precedence selects a different ansible.cfg or environment setting.
- **Quick sign:** Connection or role settings differ between directories.
- **Solution:** Inspect the active configuration and explicit overrides.
- **Useful commands:** `ansible --version`, `ansible-config dump --only-changed`

---

## 3. Host is unreachable over SSH

- **Why it happens:** Routing, firewall, port, DNS, or SSH service availability is wrong.
- **Quick sign:** Tasks fail as UNREACHABLE with timeouts or refusal.
- **Solution:** Test the same endpoint and user outside Ansible before editing tasks.
- **Useful commands:** `ssh -vvv user@host`, `ansible all -i inventory -m ansible.builtin.ping -vvv`

---

## 4. SSH authentication fails

- **Why it happens:** The remote user, private key, or agent credentials are incorrect.
- **Quick sign:** Permission denied (publickey).
- **Solution:** Verify ansible_user and key selection; use approved credentials and inspect SSH configuration.
- **Useful commands:** `ssh-add -l`, `ansible all -i inventory -m ansible.builtin.ping -u <user> --private-key <path>`

---

## 5. SSH host key verification fails

- **Why it happens:** A host is new or its stored key differs.
- **Quick sign:** Host key verification failed or identification changed.
- **Solution:** Verify the fingerprint through a trusted channel before updating known_hosts; do not globally disable checking.
- **Useful commands:** `ssh-keygen -F <host>`, `ssh-keygen -R <host>`

---

## 6. Become password is required

- **Why it happens:** Privilege escalation requires credentials that were not supplied.
- **Quick sign:** Sudo reports a password is required.
- **Solution:** Use approved escalation credentials and policy; request a become password interactively when appropriate.
- **Useful commands:** `ansible-playbook -i inventory site.yml --become --ask-become-pass`

---

## 7. Become user lacks permission

- **Why it happens:** Sudo policy, become method, or target user is incorrect.
- **Quick sign:** Privileged tasks fail despite successful SSH.
- **Solution:** Inspect sudo policy and configure the required least-privilege access.
- **Useful commands:** `sudo -l`, `ansible-config dump --only-changed`

---

## 8. Python interpreter is missing

- **Why it happens:** Most POSIX modules need remote Python, but the host lacks it or discovery selects the wrong interpreter.
- **Quick sign:** Module execution fails with Python not found.
- **Solution:** Bootstrap Python with an appropriate raw command when necessary and set ansible_python_interpreter only to a verified path.
- **Useful commands:** `ansible all -i inventory -m ansible.builtin.raw -a 'command -v python3'`

---

## 9. Required Python library is missing

- **Why it happens:** The module's execution environment lacks a dependency.
- **Quick sign:** A task reports an import or library error.
- **Solution:** Determine whether the module runs locally, remotely, or delegated; install the dependency for that interpreter.
- **Useful commands:** `ansible-doc <module>`, `python3 -m pip show <package>`

---

## 10. YAML syntax error

- **Why it happens:** Indentation, tabs, quoting, or mapping syntax is invalid.
- **Quick sign:** Parsing fails before tasks run.
- **Solution:** Check the reported area and preceding lines; use spaces and quote template expressions that begin values.
- **Useful commands:** `ansible-playbook -i inventory site.yml --syntax-check`, `yamllint site.yml`

---

## 11. Undefined variable

- **Why it happens:** A variable is absent, misspelled, or outside the current scope.
- **Quick sign:** A task fails with an undefined-variable message.
- **Solution:** Verify names, inventory groups, and variable files; use defaults only when omission is valid.
- **Useful commands:** `ansible-inventory -i inventory --graph`, `ansible-doc ansible.builtin.assert`

---

## 12. Variable value is unexpectedly overridden

- **Why it happens:** Another source has higher variable precedence.
- **Quick sign:** A play uses a different value than the inventory suggests.
- **Solution:** Trace group_vars, host_vars, role/play variables, set_fact, and extra vars; avoid duplicate ownership.
- **Useful commands:** `ansible-inventory -i inventory --host <host>`, `ansible-config dump --only-changed`

---

## 13. Template rendering fails

- **Why it happens:** Jinja syntax, variable types, filters, or lookups are invalid.
- **Quick sign:** A template task reports a rendering exception.
- **Solution:** Validate inputs and expressions, then test on a limited nonproduction target.
- **Useful commands:** `ansible-doc ansible.builtin.template`, `ansible-playbook -i inventory site.yml --check --limit <host>`

---

## 14. Loop result structure is misunderstood

- **Why it happens:** A registered looping task returns a results list rather than one ordinary result.
- **Quick sign:** Accessing a top-level stdout or rc attribute fails.
- **Solution:** Inspect non-sensitive structure and iterate over the results list; use loop_control.loop_var for nested loops.
- **Useful commands:** `ansible-doc ansible.builtin.debug`, `ansible-doc ansible.builtin.set_fact`

---

## 15. Shell operators do not work with command

- **Why it happens:** The command module does not execute through a shell.
- **Quick sign:** Pipes or redirects are treated as ordinary arguments.
- **Solution:** Prefer a dedicated module; use shell only when shell behavior is required and quote untrusted values.
- **Useful commands:** `ansible-doc ansible.builtin.command`, `ansible-doc ansible.builtin.shell`

---

## 16. Task changes on every run

- **Why it happens:** An imperative command, unstable template, or incorrect change reporting breaks idempotence.
- **Quick sign:** A second unchanged run still reports changes.
- **Solution:** Prefer state-based modules, use creates/removes where suitable, and define changed_when based on actual semantics.
- **Useful commands:** `ansible-doc ansible.builtin.command`, `ansible-playbook -i inventory site.yml --check --diff`

---

## 17. Handler does not run

- **Why it happens:** The notifying task was unchanged, the name is wrong, or a later failure prevents normal handler execution.
- **Quick sign:** Config changes occur without the expected restart.
- **Solution:** Verify notification names and change reporting; use flush_handlers or force_handlers only with a clear safety rationale.
- **Useful commands:** `ansible-playbook -i inventory site.yml --list-tasks`, `ansible-doc ansible.builtin.meta`

---

## 18. Role or collection cannot be found

- **Why it happens:** Dependencies are missing or collection/role paths differ.
- **Quick sign:** Ansible cannot resolve a role or module action.
- **Solution:** Install pinned requirements and use fully qualified module names; verify the active environment.
- **Useful commands:** `ansible-galaxy collection list`, `ansible-galaxy role list`, `ansible-galaxy collection install -r requirements.yml`

---

## 19. Vault decryption fails

- **Why it happens:** The password, vault ID, or identity configuration does not match the encrypted data.
- **Quick sign:** Ansible cannot decrypt a variable file.
- **Solution:** Use the correct approved vault identity; never commit passwords or paste decrypted secrets into logs.
- **Useful commands:** `ansible-playbook -i inventory site.yml --ask-vault-pass`, `ansible-vault --help`

---

## 20. Check mode gives misleading results

- **Why it happens:** Module support varies and later tasks may depend on results not produced during simulation.
- **Quick sign:** Tasks skip, fail, or differ from a real run.
- **Solution:** Read module check-mode support and test on a safe limited target; check mode is not a guarantee of zero side effects.
- **Useful commands:** `ansible-doc <module>`, `ansible-playbook -i inventory site.yml --check --limit <host>`

---

## Common mistakes

- disabling SSH verification to hide trust failures
- using shell for tasks with dedicated modules
- setting changed_when: false to hide real changes
- printing secrets with debug, verbosity, or diff
- assuming ping tests only ICMP; Ansible ping tests module connectivity and Python

---

## Good practices

- inspect inventory and configuration first
- use fully qualified module names
- keep automation idempotent
- limit initial runs to safe targets
- protect sensitive tasks with no_log and appropriate diff controls
- remember inventory inspection can expose sensitive variable values

---

## Must memorize

```bash
ansible --version
ansible-config dump --only-changed
ansible-inventory -i inventory --graph
ansible all -i inventory -m ansible.builtin.ping
ansible-playbook -i inventory site.yml --syntax-check
ansible-playbook -i inventory site.yml --list-hosts
ansible-playbook -i inventory site.yml --check --limit <host>
ansible-doc ansible.builtin.copy
ansible-galaxy collection list
# Verbose and diff output may expose secrets; use carefully.
```

---

## Key ideas

- Separate connection failures from task failures.
- Variable precedence and execution location explain many surprises.
- Idempotence requires correct state and change reporting.
- Check mode helps testing but does not replace safe rollout.
