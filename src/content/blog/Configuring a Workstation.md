---
title: Configuring a New Workstation for My Homelab
description: The process I used to prepare a new workstation to manage, decrypt, configure, and provision my homelab infrastructure.
pubDate: 2026-09-28
updatedDate: 2026-10-16
tags:
  - Homelab
  - Kubernetes
  - DevOps
  - Terraform
  - Ansible
  - SOPS
  - Azure
draft: false
---
## Summary

Setting up a new workstation for my homelab involves more than cloning a few repositories. The machine needs access to encrypted configuration, SSH access to the Kubernetes virtual machines, infrastructure tooling, and authenticated sessions for the services supporting the lab.

My laptop runs **Omarchy Quattro 4, which is based on Arch Linux**. This post documents the process of turning that installation into a workstation I can use to administer my infrastructure.

The goal was to restore administrative access and verify that everything worked before making infrastructure changes.

---
## What the Workstation Needs to Do

Before configuring anything, I identified the capabilities the finished workstation needed:

- Clone and update my homelab repositories over SSH.
- Connect to the Kubernetes virtual machines.
- Decrypt and edit SOPS-managed configuration.
- Authenticate to Azure for the resources that require it.
- Access the Kubernetes cluster through `kubectl`.
- Initialize Terraform against the correct remote state.
- Run Ansible against the intended hosts.

These capabilities depend on separate credentials. Git access does not automatically provide VM access, and an Azure login does not automatically authenticate Terraform to every backend or provider.

That distinction helped me organize the setup: install the tools, establish access, restore configuration, and validate each capability independently.

---
## 1. Prepare the Omarchy Toolchain

Because Omarchy is based on Arch Linux, I used its package-management tooling to prepare the workstation.

The tools required for my workflow were:

| Tool | Purpose |
|---|---|
| Git and OpenSSH | Repository access and remote administration |
| kubectl | Kubernetes administration |
| Azure CLI | Azure authentication and resource access |
| Terraform | Infrastructure provisioning |
| SOPS | Encrypted configuration |
| Ansible | Host configuration and validation |

Before installing or upgrading tools, I checked the versions required by my repositories. A new workstation does not necessarily need the newest version of every tool; it needs versions compatible with the infrastructure and automation it manages.

After installation, I checked the available versions:

```bash
git --version
ssh -V
kubectl version --client
az version
terraform version
sops --version
ansible --version
```

This gave me a baseline for comparing the laptop with my existing workstation and troubleshooting any differences.

---
## 2. Establish Git Access and Clone the Repositories

The first access check was authentication to GitHub.

GitHub authentication and VM authentication serve different purposes. I configured the GitHub SSH identity separately from the workstation-specific key used to administer my virtual machines.

To test GitHub authentication:

```bash
ssh -T git@github.com
```

On a first connection, I verified the host fingerprint before accepting it.

Once authentication worked, I cloned the repositories needed to administer the lab:

```bash
git clone git@github.com:<owner>/<homelab-repository>.git
```

I then reviewed the repository documentation, Terraform configuration, Ansible inventory, and any declared tool dependencies before continuing.

Cloning the repository restored the configuration, but the workstation still needed the credentials required to use it.

---
## 3. Create a Workstation-Specific SSH Key for the VMs

For VM administration, I created a new SSH key instead of copying the private key from my existing workstation.

A separate key gives the laptop its own identity. If I lose or retire the machine, I can revoke its access without replacing the keys used by my other devices.

```bash
ssh-keygen -t ed25519 -a 100 \
  -f ~/.ssh/homelab-laptop \
  -C "homelab-laptop"
```

I protected the private key with a passphrase and retrieved the public key:

```bash
cat ~/.ssh/homelab-laptop.pub
```

In my Terraform configuration, the VM SSH keys are managed through the `ssh_public_keys` string array in `terraform.tfvars`. Using my existing workstation, I added the laptop’s public key to that configuration.

However, applying that change through my current VM configuration would have rebooted the machines. To avoid interrupting the running lab, I also installed the public key directly on each VM through my existing workstation.

This recorded the key in the provisioning configuration while allowing me to establish access without an immediate Terraform apply.

Afterward, I tested each VM from the laptop:

```bash
ssh -i ~/.ssh/homelab-laptop <remote-user>@<vm-hostname>
```

I also added entries to `~/.ssh/config` so SSH consistently selected the intended identity:

```sshconfig
Host <vm-hostname>
  User <remote-user>
  IdentityFile ~/.ssh/homelab-laptop
  IdentitiesOnly yes
```

The host entries need to match the names or addresses used by my Ansible inventory.

---
## 4. Restore SOPS Decryption Access

My repositories contain encrypted configuration managed with SOPS. Cloning those files does not grant access to their contents.

The workstation also needs access to the decryption identity configured for those files. For files encrypted to my age recipient, that means securely restoring the corresponding age private key.

I prepared the local key directory with restrictive permissions:

```bash
mkdir -p ~/.config/sops/age
chmod 700 ~/.config/sops/age
```

After securely restoring the key into `keys.txt`, I restricted access:

```bash
chmod 600 ~/.config/sops/age/keys.txt
```

When an explicit key-file setting is needed:

```bash
export SOPS_AGE_KEY_FILE="$HOME/.config/sops/age/keys.txt"
```

The path can be documented; the private key contents must remain protected.

I tested decryption against a known encrypted file:

```bash
sops decrypt <encrypted-file-path> >/dev/null
```

This checked that SOPS could decrypt the file without printing the plaintext to the terminal or saving it as a separate file.

Files using another SOPS backend require access to that backend as well.

---
## 5. Authenticate to Azure

Azure authentication is another part of the workstation setup because my lab uses Azure-backed resources, including Key Vault.

I signed in and explicitly selected the intended subscription:

```bash
az login
az account list --output table
az account set --subscription "<subscription-name-or-id>"
az account show --output table
```

For a session without a usable browser, device-code authentication is an alternative:

```bash
az login --use-device-code
```

I checked access to the expected vault:

```bash
az keyvault show \
  --name <vault-name> \
  --output table
```

Where my permissions allowed it, I also checked secret metadata:

```bash
az keyvault secret list \
  --vault-name <vault-name> \
  --query "[].name" \
  --output table
```

These checks validate metadata access. They do not establish that the identity can retrieve every secret or perform every required cryptographic operation.

The relevant application or decryption workflow still needs its own validation.

---
## 6. Configure Kubernetes Access

With the tools and network access available, I restored the kubeconfig required to reach my cluster.

A kubeconfig can contain credentials, so I kept it outside the repository and restricted access to it:

```bash
mkdir -p ~/.kube
chmod 700 ~/.kube
```

After securely placing the configuration at `~/.kube/config`:

```bash
chmod 600 ~/.kube/config
```

Before interacting with the cluster, I inspected the available contexts:

```bash
kubectl config get-contexts
kubectl config use-context <homelab-context>
kubectl config current-context
```

Then I performed read-only checks:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl auth can-i get pods --all-namespaces
```

These checks helped confirm that the laptop could reach the cluster, authenticate, and perform the expected administrative reads.

Checking the context explicitly is a habit I want to maintain as I add more environments.

---
## 7. Initialize and Validate Terraform

Terraform requires more than an installed binary. It also needs the correct configuration, backend access, workspace, variables, and provider credentials.

From the repository’s Terraform root directory:

```bash
cd <repository-path>/terraform

terraform fmt -check -recursive
terraform init
terraform workspace show
terraform validate
terraform plan
```

Backend authentication and provider authentication are separate concerns. An Azure CLI session may support an Azure provider workflow, but it does not automatically grant access to a different remote-state service or to my Proxmox infrastructure.

I checked those credentials according to the repository’s configuration before reviewing the plan.

The first plan from a new workstation is particularly useful. I compared its proposed changes with what I expected from the current repository and infrastructure.

Unexpected replacements, deletions, or widespread changes would be a reason to investigate before continuing.

I did not need to run `terraform apply` simply to prove that Terraform was installed correctly. Initialization, validation, and a reviewed plan provided the checks I needed.

---
## 8. Validate Ansible Access

Next, I checked that Ansible could load the inventory and connect using the laptop’s SSH identity.

```bash
cd <repository-path>/ansible

ansible-inventory \
  --inventory <inventory-path> \
  --graph
```

If the repository declares collections or roles, those dependencies also need to be installed:

```bash
ansible-galaxy collection install \
  -r <collections-requirements-file>

ansible-galaxy role install \
  -r <roles-requirements-file>
```

I then tested the Kubernetes VM group:

```bash
ansible <kubernetes-group> \
  --inventory <inventory-path> \
  --module-name ping
```

Ansible’s `ping` module checks remote execution and Python availability over the configured connection. It is different from an ICMP network ping.

For playbooks that support check mode, I could then preview their behavior:

```bash
ansible-playbook \
  --inventory <inventory-path> \
  <playbook-path> \
  --check
```

Check mode has limits, so I treat its output as an additional review step. I only enable diff output where the tasks are configured to avoid exposing sensitive values.

---
## Validation Checklist

Before considering the laptop ready, I checked each capability:

- [ ] GitHub authentication works with the configured Git identity.
- [ ] The laptop’s public SSH key is recorded in the VM provisioning configuration.
- [ ] SSH access works to every required VM.
- [ ] SOPS can decrypt a known encrypted file.
- [ ] Azure CLI targets the intended tenant and subscription.
- [ ] The required Azure resource access works.
- [ ] kubectl targets the correct context and can list cluster nodes.
- [ ] Terraform initializes against the intended backend and workspace.
- [ ] Terraform produces a plan I understand.
- [ ] Ansible loads the inventory and successfully pings the VM group.
- [ ] Pending Git changes contain no credentials or decrypted configuration.

For the final repository review:

```bash
git status --short
git diff
git diff --cached
```

I also kept local credential files outside the repository. Git status alone cannot establish that a directory contains no sensitive files, particularly when files are ignored.

---
## What I Learned

The most useful part of this process was seeing how many independent pieces make an administration workstation functional.

The repositories hold the configuration, but using that configuration depends on SSH identities, decryption keys, cloud sessions, Kubernetes credentials, and remote-state access.

The SSH key update also exposed a practical improvement for my lab: routine access changes should have a configuration-management path that does not require rebooting the VMs. Recording the key in Terraform and installing it manually worked for this migration, but I want future workstation onboarding to be more repeatable.

My next step is to automate the nonsecret parts of the workstation setup: tool installation, repository checkout, configuration templates, and validation commands. Credential recovery and enrollment can remain explicit steps.

That would make the next migration faster while preserving a clear understanding of what grants the workstation access.