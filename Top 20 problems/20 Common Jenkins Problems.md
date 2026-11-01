---
tags:
  - Jenkins
  - Troubleshooting
---

Knowing these problems helps you diagnose Jenkins builds, agents, plugins, and pipeline failures safely.

---

## Main idea

Most Jenkins problems involve controller health, plugins, agents, credentials, pipeline syntax, artifacts, or resource pressure.

Commands below assume a systemd Linux installation with the default service names. Jenkins containers, custom `JENKINS_HOME` locations, and externally managed agents can use different service names and paths.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Jenkins UI is unreachable

- **Why it happens:** The controller is stopped, the port mapping is wrong, a reverse proxy is misconfigured, or a firewall blocks access.
- **Quick sign:** Browser requests time out or return a proxy error.
- **Solution:** Check the controller process and logs, verify the configured port and reverse proxy, then test access from the expected network path.
- **Useful commands:** `systemctl status jenkins`, `journalctl -u jenkins --no-pager`, `ss -lntp`, `curl -v http://127.0.0.1:8080`

---

## 2. Jenkins fails to start

- **Why it happens:** Java is missing or incompatible, disk space is exhausted, configuration is invalid, or a plugin prevents startup.
- **Quick sign:** The service enters a failed state or repeatedly stops.
- **Solution:** Read startup logs, verify the supported Java version, free disk space, and disable a recently added plugin only through the documented recovery procedure.
- **Useful commands:** `journalctl -u jenkins --no-pager`, `java -version`, `df -h`, `systemctl status jenkins`

---

## 3. Jenkins runs out of disk space

- **Why it happens:** Build logs, workspaces, artifacts, plugin downloads, or old builds accumulate.
- **Quick sign:** Builds fail with no-space errors or the UI becomes slow.
- **Solution:** Identify the largest directories, configure build retention and artifact policies, and archive or clean only verified disposable data.
- **Useful commands:** `du -sh /var/lib/jenkins/*`, `df -h`, `find /var/lib/jenkins/jobs -type d -name builds`

---

## 4. Plugin installation or update fails

- **Why it happens:** Update-site connectivity, incompatible plugin versions, Java version constraints, or controller disk space cause failure.
- **Quick sign:** Plugin manager reports download or dependency errors.
- **Solution:** Check controller logs and update-site access, verify compatibility, and update plugins in controlled batches with a backup.
- **Useful commands:** `journalctl -u jenkins --no-pager`, `curl -I <update-site-url>`, `df -h`

---

## 5. Jenkins is slow or unresponsive

- **Why it happens:** Heap pressure, excessive concurrent builds, plugin overhead, slow storage, or too many queued jobs affect the controller.
- **Quick sign:** UI requests and builds become slow while CPU, memory, or disk usage rises.
- **Solution:** Inspect controller metrics and logs, reduce controller workload, tune JVM settings according to supported guidance, and move heavy work to agents.
- **Useful commands:** `top`, `free -m`, `iostat 1`, `journalctl -u jenkins --no-pager`

---

## 6. Build stays queued

- **Why it happens:** No suitable agent is online, labels do not match, executor slots are full, or a lock/resource blocks the job.
- **Quick sign:** The job remains in the queue without starting.
- **Solution:** Inspect queue details, agent labels, executor usage, and node availability; adjust labels or capacity only after confirming the requirement.
- **Useful commands:** Jenkins UI: Manage Jenkins, Nodes, Build Queue; `curl -s http://<controller>/queue/api/json`

---

## 7. Agent is offline

- **Why it happens:** The agent process stopped, connectivity failed, Java is unavailable, or authentication/token configuration is invalid.
- **Quick sign:** The node appears offline and builds cannot be dispatched to it.
- **Solution:** Check agent logs and connectivity, verify Java and agent configuration, then restart the agent through the approved method.
- **Useful commands:** `ssh <agent-host>`, `java -version`, `journalctl -u jenkins-agent --no-pager`

---

## 8. Agent cannot connect to controller

- **Why it happens:** The controller URL, inbound agent port, proxy, firewall, or credentials are wrong.
- **Quick sign:** Agent logs show connection refused, timeout, or authentication failure.
- **Solution:** Verify the controller URL and agent protocol/port, test network reachability, and reconfigure the agent using current credentials.
- **Useful commands:** `curl -v http://<controller>:8080`, `ss -lntp`, `journalctl -u jenkins-agent --no-pager`

---

## 9. Wrong agent runs the job

- **Why it happens:** The pipeline label expression matches an unintended node or no label constraint is set.
- **Quick sign:** A build runs on an unexpected agent or fails because required tools are absent.
- **Solution:** Define precise labels and verify agent availability before changing pipeline configuration.
- **Useful commands:** Jenkins UI: Nodes; `curl -s http://<controller>/computer/api/json`

---

## 10. Pipeline fails at checkout

- **Why it happens:** Repository URL, branch, credentials, SSH key, or network access is incorrect.
- **Quick sign:** The stage fails before build steps with Git authentication or fetch errors.
- **Solution:** Verify the repository reference and credential binding; test access with the same identity from the executing agent.
- **Useful commands:** `git ls-remote <repo-url>`, `git -C workspace status`, `ssh -T git@github.com`

---

## 11. Credentials are missing or invalid

- **Why it happens:** A credential ID is wrong, deleted, scoped incorrectly, or the secret has expired.
- **Quick sign:** A stage reports credential not found or authentication failure.
- **Solution:** Confirm the exact credential ID and its scope; rotate expired secrets and avoid printing credential values.
- **Useful commands:** Jenkins UI: Manage Jenkins, Credentials; inspect pipeline logs without enabling secret masking bypass

---

## 12. Environment variable is empty

- **Why it happens:** The variable is defined in the wrong scope, set after use, or masked by a tool-specific environment.
- **Quick sign:** A command receives an empty value or uses an unexpected default.
- **Solution:** Print only known non-secret variables at the point of use and verify pipeline/environment scoping.
- **Useful commands:** `sh 'printf "%s\\n" "$MY_NON_SECRET_VAR"'`, `sh 'command -v <tool>'`

---

## 13. Pipeline syntax error

- **Why it happens:** Declarative syntax, missing blocks, incorrect quoting, or unsupported directives are used.
- **Quick sign:** The pipeline fails immediately with a syntax or validation error.
- **Solution:** Validate the Jenkinsfile, inspect the reported line context, and test a minimal version of the failing stage.
- **Useful commands:** Jenkins UI: Pipeline Syntax; `git diff -- Jenkinsfile`

---

## 14. Shell command works locally but fails in Jenkins

- **Why it happens:** PATH, working directory, user, tool versions, or shell behavior differ on the agent.
- **Quick sign:** A command succeeds interactively but fails during the build.
- **Solution:** Run the command with the same user and working directory, then use explicit tool paths or a tool installation where appropriate.
- **Useful commands:** `pwd`, `whoami`, `printf '%s\\n' "$PATH"`, `command -v <tool>`

---

## 15. Docker commands fail inside a build

- **Why it happens:** Docker is unavailable on the agent, the user lacks daemon access, or the build uses an incompatible Docker integration.
- **Quick sign:** Docker build or run steps fail with daemon or permission errors.
- **Solution:** Verify Docker availability and agent permissions; use the organization's approved container-build pattern rather than granting broad host access.
- **Useful commands:** `docker version`, `docker info`, `id`, `systemctl status docker`

---

## 16. Build artifacts are missing

- **Why it happens:** The artifact path is wrong, the build did not produce output, or archival/archive-artifacts settings are incorrect.
- **Quick sign:** The job succeeds but no artifacts appear.
- **Solution:** Confirm the actual output location and archive pattern; test the path from the workspace.
- **Useful commands:** `find . -maxdepth 3 -type f`, `ls -la target/`, `ls -la build/`

---

## 17. Workspace is dirty or contains old files

- **Why it happens:** Previous builds left generated files, stale caches, or conflicting state.
- **Quick sign:** A clean local build works while the Jenkins build behaves differently.
- **Solution:** Clean the workspace deliberately, inspect generated paths, and make the pipeline reproducible rather than relying on hidden state.
- **Useful commands:** `git status`, `git clean -nxd`, `find . -maxdepth 2 -type d`

---

## 18. Concurrent builds conflict

- **Why it happens:** Two builds use the same workspace, port, database, or external resource at once.
- **Quick sign:** Builds fail intermittently with file, port, or resource conflicts.
- **Solution:** Prevent concurrent execution where needed, isolate workspaces, and parameterize external resources.
- **Useful commands:** Jenkins UI: job configuration; `ss -lntp`, `ps -ef`

---

## 19. Webhook does not trigger a build

- **Why it happens:** The webhook URL, token, repository settings, network path, or plugin configuration is wrong.
- **Quick sign:** Pushes occur but no build starts.
- **Solution:** Check webhook delivery logs, verify the endpoint and authentication, and test the trigger with a controlled commit.
- **Useful commands:** `curl -I http://<controller>/github-webhook/`, Jenkins UI: job build history

---

## 20. Build hangs with no output

- **Why it happens:** A command waits for input, a network call stalls, a lock is held, or a child process keeps the stage alive.
- **Quick sign:** The stage remains running without progressing.
- **Solution:** Inspect the build console and process tree, add appropriate timeouts, and identify the waiting command before killing processes.
- **Useful commands:** `ps -ef --forest`, `ss -lntp`, `timeout <seconds> <command>`

---

## Common mistakes

- running production deployments from an unsecured controller
- storing secrets in Jenkinsfiles or build logs
- updating all plugins at once without testing
- granting agents unnecessary controller privileges
- treating a successful build as proof that deployment succeeded

---

## Good practices

- keep the controller lightweight and delegate builds to agents
- use precise agent labels and reproducible tool versions
- configure build retention and artifact policies
- use credentials bindings and avoid echoing secrets
- add timeouts to network-dependent stages
- back up Jenkins configuration before major plugin changes

---

## Must memorize

```bash
systemctl status jenkins
journalctl -u jenkins --no-pager
java -version
df -h
du -sh /var/lib/jenkins/*
ss -lntp
curl -v http://127.0.0.1:8080
docker version
```

---

## Key ideas

- Controller health, agent availability, and job configuration are separate layers.
- A queued build usually means scheduling or capacity, not a failed build.
- Agent environment differences explain many locally working commands failing in Jenkins.
- Protect credentials and avoid exposing them in logs.
