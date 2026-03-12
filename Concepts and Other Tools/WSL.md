---
tags:
  - Other_Tools
---
WSL, or Windows Subsystem for Linux, lets you run a full Linux distro (like Ubuntu) natively on Windows without VMs or dual-boot. WSL 2 uses a lightweight VM for better performance and full kernel support.

## WSL 1 Architecture

Translates Linux syscalls (e.g., fork, exec) to Windows NT equivalents via kernel drivers (lxcore.sys, lxss.sys). No VM—direct file/network sharing with Windows, but limited compatibility for some binaries. LXSS Manager launches distros as "pico processes."[](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/cmdline/wsl-architectural-overview)​

## WSL 2 Architecture

Runs genuine Linux kernel in a managed Hyper-V VM with ext4 VHDX disk. Shares CPU/memory/network; Windows files accessible via /mnt/c (9P protocol). Faster I/O, Docker Desktop integration, and GPU support for ML. Init handles namespaces (PID, mount, user).
## DevOps Benefits

Install tools like Docker, Kubernetes (minikube), Ansible, Terraform, AWS CLI directly—ideal for your Windows setup while practicing Linux/Bash for CKAD/AWS. Run Linux containers alongside Windows ones, script pipelines, and debug EKS locally without corporate VM restrictions. Faster than full VMs for daily automation.

## Limitations

File I/O slower across Windows/Linux; some networking quirks with Docker Desktop. Heavy K8s clusters may need native Linux, but great starter for your Udemy labs.[](https://dev.to/david_oyewole/why-i-switched-from-wsl-to-ubuntu-for-devops-a-personal-journey-44dp)​

## Interview Answer Script

"WSL enables native Linux on Windows for DevOps workflows. With WSL 2 (`wsl --install`), I run Ubuntu, Docker, and kubectl seamlessly—e.g., `wsl -d Ubuntu docker run nginx`. It accelerates CI/CD testing without VMs, though I mount /mnt/c carefully for perf." Demo via `wsl --list --verbose`