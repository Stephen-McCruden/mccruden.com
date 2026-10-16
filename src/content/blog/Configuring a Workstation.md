---
title: Configuring a New Workstation for My Homelab
description: The process I used to prepare a new workstation to manage, decrypt, configure, and provision my homelab infrastructure.
pubDate: 2026-08-28
updatedDate: 2026-08-28
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

Setting up a new workstation for my homelab involves more than cloning a few repositories. The workstation needs access to encrypted configuration, permission to connect to the Kubernetes virtual machines, the same infrastructure tooling as my previous machine, and authenticated sessions for the cloud services that back my secrets and Terraform state.

This post documents the full process I followed to turn a clean workstation into a trusted homelab administration machine.

---

## What the Workstation Needs to Do

Before installing anything, I wrote down the capabilities the finished workstation needed:

- Clone and work with my homelab repositories over SSH.
- Decrypt and edit SOPS-managed secrets.
- Authenticate to Azure for Key Vault access and Terraform provisioning.
- Run `kubectl` against my Kubernetes clusters.
- Run Terraform plans and applies using the correct remote state and credentials.
- Run Ansible against the Kubernetes virtual machines.
- Connect to those virtual machines over SSH using a workstation-specific key.

That list gave me a useful order of operations: establish identity first, install the tools second, authenticate third, and validate access before making any infrastructure changes.

---
## 1. Create a Workstation-Specific SSH Key

I created a new SSH key for this workstation rather than copying the private SSH key from another machine. Giving each workstation its own identity makes rotation and revocation much simpler if a machine is lost or retired.

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/<homelab-key-name> -C "<workstation-name>"
```

I then added the public key—not the private key—to the authorized keys managed for each Kubernetes VM. In my environment, that means updating `<Ansible inventory, cloud-init configuration, or repository path>` and applying the change through `<the normal configuration-management workflow>`.

```bash
cat ~/.ssh/<homelab-key-name>.pub
```

After the authorized key was deployed, I tested each host explicitly:

```bash
ssh -i ~/.ssh/<homelab-key-name> <user>@<kubernetes-vm>
```

I also added a narrowly scoped entry to `~/.ssh/config` so that Ansible and my own SSH sessions consistently use the new key:

```sshconfig
Host <homelab-host-pattern>
  User <remote-user>
  IdentityFile ~/.ssh/<homelab-key-name>
  IdentitiesOnly yes
```

To distribute the new public key, I added it to `terraform.tfvars` in the `ssh_public_keys` string array on my existing workstation. I then also manually moved the public key over to each VM via my existing workstation as running a `terraform apply` would reboot each VM and cause downtime.

---
## 2. Retrieve My SOPS Private Key Securely

My homelab repositories contain secrets encrypted with SOPS. A clean workstation can clone those files, but it cannot decrypt them until it has the corresponding private key.

I retrieved the SOPS private key from `<offline backup, password manager, removable media, or other secure recovery location>`. I transferred it directly to the expected local key store with restrictive permissions. I did not send it through chat or email, paste it into a repository, or include its contents in shell history.

```bash
mkdir -p <sops-key-directory>
chmod 700 <sops-key-directory>
# Securely place the recovered key at <sops-key-path>.
chmod 600 <sops-key-path>
```

Depending on the SOPS backend, I may also need to set an environment variable such as `<SOPS key-file environment variable>` or configure the appropriate key store. The exact path and variable belong in my private workstation documentation, not in this public post.

I validated the key with a read-only decrypt operation against a known encrypted file:

```bash
sops decrypt <path-to-encrypted-test-file> >/dev/null
```

That proved the key worked without writing decrypted secrets to disk or printing them.

---
## 3. Install the Infrastructure Toolchain

Next I installed the command-line tools used throughout the homelab:

- `kubectl` for Kubernetes administration.
- Azure CLI (`az`) for Azure authentication and Key Vault access.
- Terraform for provisioning and managing infrastructure.
- SOPS for encrypted configuration.
- Ansible for configuring the Kubernetes VMs and other hosts.

```bash
# Replace these placeholders with commands for the specific workstation's OS.
<install kubectl>
<install Azure CLI>
<install Terraform>
<install SOPS>
<install Ansible>
```

I prefer the vendors' signed package repositories or a version manager where practical. After installation, I captured the versions so I could compare them with the versions used by CI and my previous workstation:

```bash
kubectl version --client
az version
terraform version
sops --version
ansible --version
```

---
## 4. Authenticate to Azure

Azure serves two separate needs in this workflow: access to Key Vault and credentials for Terraform provisioning. I authenticated with the Azure CLI and selected the intended tenant and subscription rather than trusting whatever default happened to be active.

```bash
az login
az account list --output table
az account set --subscription "<subscription-name-or-id>"
az account show --output table
```

If the workstation has no usable browser, the device-code login flow is an alternative:

```bash
az login --use-device-code
```

I then checked that my identity could reach the expected Key Vault without exposing any secret values:

```bash
az keyvault show --name <vault-name> --output table
az keyvault secret list --vault-name <vault-name> --query "[].name" --output table
```

For Terraform, the Azure CLI session can provide local development credentials. I still verified the active subscription before running Terraform, because a valid login to the wrong subscription is more dangerous than a failed login.

---
## 5. Configure Kubernetes Access

With `kubectl` installed, I restored or regenerated my kubeconfig using `<the cluster bootstrap or configuration process>`. I did not commit the kubeconfig because it can contain credentials and internal endpoint information.

Before running any mutating command, I inspected the available contexts and selected the homelab context explicitly:

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

---
## 6. Initialize and Validate Terraform

I cloned the infrastructure repository and moved to `<Terraform root module path>`. Before planning, I confirmed Azure was targeting the correct subscription and that the expected backend configuration was available.

```bash
git clone <homelab-repository-url>
cd <terraform-directory>
terraform fmt -check -recursive
terraform init
terraform validate
terraform plan
```

The first plan from a new workstation is an important safety check. My expected result was `<no changes, or a description of the expected changes>`. An unexpected large plan would indicate a credential, backend, workspace, provider, or configuration problem and would stop the setup process before any apply.

```bash
terraform workspace show
```

I did not run `terraform apply` merely to test the installation. A successful initialization, validation, and reviewed plan were enough to prove that the workstation could participate in the provisioning workflow.

---
## 7. Validate Ansible Against the Kubernetes VMs

Finally, I checked that Ansible could load my inventory and use the new SSH identity:

```bash
cd <ansible-directory>
ansible-inventory --inventory <inventory-path> --graph
ansible <kubernetes-group> --inventory <inventory-path> --module-name ping
```

If the repository uses an `ansible.cfg`, collections, or roles, I also installed those declared dependencies:

```bash
ansible-galaxy collection install -r <collections-requirements-file>
ansible-galaxy role install -r <roles-requirements-file>
```

The ping module confirmed SSH connectivity and Python availability without changing the remote systems. Once that passed, I used the repository's check-mode workflow where supported:

```bash
ansible-playbook --inventory <inventory-path> <playbook-path> --check --diff
```

---

## Validation Checklist

At the end of the setup, I verified each capability independently:

- [ ] The workstation can authenticate to the Git host using its own SSH key.
- [ ] The new public SSH key is deployed to every required Kubernetes VM.
- [ ] SOPS can decrypt a known file without writing plaintext to disk.
- [ ] `kubectl` uses the correct context and can list the cluster nodes.
- [ ] Azure CLI is logged in to the intended tenant and subscription.
- [ ] The Azure identity can access the required Key Vault metadata.
- [ ] Terraform can initialize its backend, validate the configuration, and produce the expected plan.
- [ ] Ansible can parse the inventory and ping the Kubernetes VM group.
- [ ] No private keys, decrypted files, kubeconfigs, Terraform state, or tokens appear in Git status.

```bash
git status --short
```

---
## What I Learned

The most important part of building an administration workstation is not installing binaries. It is re-establishing trust in a deliberate, auditable way.

Separate SSH keys make workstation access revocable. A protected SOPS recovery process makes encrypted Git repositories useful without turning the private key into another unmanaged secret. Explicit Azure subscription and Kubernetes context checks reduce the chance of operating on the wrong environment. Read-only Terraform and Ansible validation proves the toolchain works before the new workstation is allowed to change infrastructure.
