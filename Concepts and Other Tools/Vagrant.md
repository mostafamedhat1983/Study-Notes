---
tags:
  - Other_Tools
---
**Vagrant** = HashiCorp tool for **reproducible VM environments** via code (eliminates "works on my machine")

## 🎯 **Core Concept**



`Vagrant (CLI + Vagrantfile) → Provider (virtualbox/hyperv) → Hypervisor → VMs`

**NOT a hypervisor** - orchestrates **existing** hypervisors via providers

## 🏗️ Architecture**



┌─────────────────────┐   Vagrantfile (Ruby DSL)
│    Vagrant CLI      │          ← vagrant up/down/ssh
├─────────────────────┤
│   Vagrant Provider         │   virtualbox / hyperv / aws / docker
│   (Plugin Layer)              │   ← Translates Vagrant commands
├─────────────────────┤
│   Hypervisor                  │   VirtualBox / Hyper-V / ESXi / KVM
│   (Virtualization)            │   ← Creates actual VMs
└─────────────────────┘


## 🔍 **Key Distinction**

| Layer          | What it does                                           | Examples                                |
| -------------- | ------------------------------------------------------ | --------------------------------------- |
| **Provider**   | **Vagrant's plugin/adapter** that speaks to hypervisor | `virtualbox`, `hyperv`, `vmware`, `aws` |
| **Hypervisor** | **Actual virtualization engine** creating VMs          | VirtualBox, Hyper-V, VMware ESXi, KVM   |

## ⚡ **Essential Commands**


vagrant init ubuntu/jammy64      # Setup project
vagrant up                      # Create/start VMs
vagrant ssh                     # Enter VM
vagrant status                  # Check states
vagrant halt                    # Graceful stop
vagrant destroy                 # Clean slate
vagrant box add <box>           # Download images
vagrant box list                # List local boxes
