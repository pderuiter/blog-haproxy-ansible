# blog-haproxy-ansible

Companion code for the blog post
[Automating HAProxy and Data Plane API Deployment with Ansible](https://blog.bsdserver.nl/automating-haproxy-3-1-and-data-plane-api-deployment-with-ansible/).

The post's playbook, split into two roles: HAProxy from the official HAProxy
packages (the vbernat PPA on Ubuntu, haproxy.debian.net on Debian) and the
HAProxy Data Plane API 3.x from the official release, with the post's
`haproxy.cfg`, `dataplaneapi.yaml` and systemd unit.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | One Ubuntu 24.04 VM on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/haproxy_install` | Signing key, deb822 repository and the `haproxy` package for the chosen branch |
| `ansible/roles/dataplaneapi` | `haproxy.cfg`, the Data Plane API binary, its configuration and systemd unit |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VM)
- ansible-core >= 2.15 (tested with the current ansible-core release)
- Target host: Ubuntu 24.04 (tested) or Debian 13 (repository tasks tested), amd64, with Internet access

## Quick start

```bash
# 1. Machine (optional; skip if you already have a host)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. HAProxy and the Data Plane API
cd ../ansible
cp inventory.example.yml inventory.yml         # if you skipped Terraform
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
openssl passwd -6 'your-secure-password'      # put the hash in vault.yml
ansible-vault encrypt group_vars/all/vault.yml
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass

# 3. Test (the API listens on localhost only)
ssh -L 5555:127.0.0.1:5555 ansible@10.0.0.10 -N &
curl -s -u admin:<password> http://127.0.0.1:5555/v3/info
```

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in each role's `defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `haproxy_install_version` | `3.3` | HAProxy branch. 3.3 is the newest the PPA builds for Ubuntu 24.04 (maintained until about Q1 2027); `3.2` is LTS on 24.04, `3.4` (LTS) needs Ubuntu 26.04 or Debian 13 |
| `dataplaneapi_version` | `3.4.7` | Data Plane API release. Changing it upgrades (or downgrades) an existing install |
| `dataplaneapi_user` | `admin` | API user in the `dataplaneapi` userlist |
| `vault_dataplaneapi_password_hash` | (vault) | SHA-512 crypt hash of the API password (`openssl passwd -6`), written as `password` in the userlist |
| `dataplaneapi_bind_address` / `dataplaneapi_port` | `127.0.0.1` / `5555` | Where the API listens. Plain HTTP, so localhost only |
| `dataplaneapi_bootstrap_marker` | `/etc/haproxy/.haproxy-cfg-bootstrapped` | Written after the first `haproxy.cfg`; delete it to write the template again |
| `dataplaneapi_config_file` | `/etc/haproxy/dataplaneapi.yaml` | Data Plane API configuration |

The `dataplaneapi.yaml` template is written in the normalized form the Data
Plane API rewrites on every start, so restarting the service does not make the
next run report a change.

## Ansible or the API

`haproxy.cfg` is bootstrap-only. The first run replaces the package's default
file with the template, reloads HAProxy and writes a marker file. After that
the template task runs with `force: false` and the Data Plane API owns
`haproxy.cfg`: later runs leave changes made through the API alone. Delete
`/etc/haproxy/.haproxy-cfg-bootstrapped` to write the template again (this
overwrites the API's changes).

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container: the first
run installs HAProxy 3.3.17 and the Data Plane API 3.4.7, the second run
reports `changed=0`, and so does a run after restarting both services.
A server added through the API is still in `haproxy.cfg` after the next run
(`changed=0`, HAProxy not restarted). The API accepts the hashed password and
rejects a wrong one, and is not reachable from other hosts.
`/v3/info` answers and a v3 transaction that adds a server ends up in
`haproxy.cfg`. An upgrade from Data Plane API 3.3.12 to 3.4.7 reports the
download and a restart, and the run after it `changed=0`. ansible-lint
(production profile), yamllint, `terraform validate` and tflint are clean. The
Terraform code has not been applied against a real Proxmox cluster.

## License

MIT
