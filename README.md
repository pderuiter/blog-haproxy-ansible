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
ansible-vault encrypt group_vars/all/vault.yml # after setting the password
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass

# 3. Test
curl -s -u admin:<password> http://10.0.0.10:5555/v3/info
```

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in each role's `defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `haproxy_install_version` | `3.3` | HAProxy branch. 3.3 is the newest the PPA builds for Ubuntu 24.04 (maintained until about Q1 2027); `3.2` is LTS on 24.04, `3.4` (LTS) needs Ubuntu 26.04 or Debian 13 |
| `dataplaneapi_version` | `3.4.7` | Data Plane API release. Changing it upgrades (or downgrades) an existing install |
| `dataplaneapi_user` | `admin` | API user in the `dataplaneapi` userlist |
| `vault_dataplaneapi_password` | (vault) | API password, written as `insecure-password` as in the post |
| `dataplaneapi_bind_address` / `dataplaneapi_port` | `0.0.0.0` / `5555` | Where the API listens |
| `dataplaneapi_config_file` | `/etc/haproxy/dataplaneapi.yaml` | Data Plane API configuration |

The `dataplaneapi.yaml` template is written in the normalized form the Data
Plane API rewrites on every start, so restarting the service does not make the
next run report a change.

## Ansible or the API

The Data Plane API changes `haproxy.cfg` itself. A later run of this playbook
writes the template back and undoes those changes. Use the playbook for the
initial install and upgrades, and either keep the API changes in the template
or stop running the `haproxy.cfg` task once the API owns the configuration.

## Tested

`site.yml` was run against a fresh Ubuntu 24.04 systemd container: the first
run installs HAProxy 3.3.17 and the Data Plane API 3.4.7, the second run
reports `changed=0`, and so does a run after restarting both services.
`/v3/info` answers and a v3 transaction that adds a server ends up in
`haproxy.cfg`. An upgrade from Data Plane API 3.3.12 to 3.4.7 reports the
download and a restart, and the run after it `changed=0`. ansible-lint
(production profile), yamllint, `terraform validate` and tflint are clean. The
Terraform code has not been applied against a real Proxmox cluster.

## License

MIT
