---
title: Configuration
weight: 2
---

## Terraform variables

Set them in `terraform.tfvars` (next to where you run slurmstack, git-ignored)
or point `-var-file` elsewhere. Defaults are from `terraform/variables.tf`.

| Variable | Default | Meaning |
|---|---|---|
| `allowed_ssh_cidrs` | *(required)* | Source CIDRs allowed to reach the frontend on port 22. |
| `cluster_name` | `slurm` | Prefix of every resource and the Slurm `ClusterName`. Lowercase, `[a-z0-9-]`, at most 21 characters. |
| `image_name` | `ubuntu-2404.amd64-genericcloud.20260108` | Glance image of every node. The roles target Ubuntu 24.04. |
| `image_visibility` | `public` | Only an image with this visibility is accepted; `null` accepts any. |
| `ssh_user` | `ubuntu` | Login of the image. |
| `frontend_flavor` | `vd.16` | Flavor of the frontend (slurmctld, login, and where Spack builds). |
| `worker_flavor` | `vd.32` | Flavor of each worker (32 vCPUs, 64 GB). Check the project's quota covers `worker_count` of them. |
| `worker_count` | `4` | Number of workers, 1 to 64. |
| `nfs_flavor` | `vd.2` | Flavor of the NFS server. |
| `nfs_volume_size` | `50` | GB of the Cinder volume behind `/home`; `0` keeps it on the NFS server's root disk. |
| `apps_volume_size` | `100` | GB of the Cinder volume behind `/opt/apps` (the shared Spack instance); `0` keeps it on the NFS server's root disk. |
| `nfs_volume_type` | `null` | Cinder volume type, `null` for the cloud's default. Prefer an encrypted type where it works; VirtualData's `encrypted` is currently refused ("Invalid encryption spec"). |
| `external_network_name` | `public-2` | Network the router gateway and the floating IP come from. |
| `private_network_name` | `slurm-net` | Name of the private network. |
| `private_subnet_cidr` | `10.42.0.0/24` | CIDR of the private subnet. |
| `dns_nameservers` | `["8.8.8.8", "1.1.1.1"]` | Resolvers given by the subnet's DHCP. |
| `router_name` | `slurm-router` | Name of the router. |
| `ssh_public_key_path` | `~/.ssh/id_ed25519.pub` | Public key installed on every node. |

## Users

`users.yaml` (next to where you run slurmstack, git-ignored; start from
`users.yaml.example`) lists the people glauth publishes. SSSD makes them
Linux users on every node, and their home is created on the NFS server.

```yaml
motd: |                    # optional: shown at every login
  Welcome to the course's Slurm cluster.
admin_ssh_keys:            # optional: more keys for the admin account
  - ssh-ed25519 AAAA… me@desktop
users:
  - name: alice
    uid: 20001
    ssh_keys:
      - ssh-ed25519 AAAA… alice@laptop
  - name: bob
    uid: 20002
    role: operator         # optional: see Operators below
    ssh_keys:
      - ssh-ed25519 AAAA… bob@laptop
```

- `uid` is written, never assigned: it owns the files in `/home`. Range
  20000–29999. When someone leaves, add their uid to `retired_uids:` (a
  top-level list): it is then refused for anyone else.
- `role`: left out, a user; `operator`, a Slurm operator. Nothing else is
  accepted.
- `motd` is the message of the day: shown at every interactive login on
  the frontend and the workers (not to `scp`, `sftp` or `ssh host
  command`), followed by a line stating the job limits, generated from
  `ansible/vars/slurm.yml` (`motd_limits` in the motd role; `""` drops
  it). Ubuntu's own login messages (news, adverts, system information,
  pending updates) are turned off. Plain text up to 4000 bytes: template
  delimiters (`{{`, `{%`, `{#` and their closers) and control characters
  are refused, since go-ansible would evaluate the former and the latter
  would drive every user's terminal. A user can hide it with
  `~/.hushlogin`.
- `admin_ssh_keys` open the admin account (`ubuntu`) besides the key of
  Terraform's keypair, on every node. **Each is root on the whole
  cluster** (the admin has sudo without a password) and logs in as
  `ubuntu`, so the logs cannot tell who did what: give colleagues an
  account of their own, an operator if they need one. Terraform's key is
  kept apart and never rewritten, so emptying the list cannot lock you out.
- Only the key itself is kept; its comment and any options are dropped.
- Logins are by SSH key only. There are no passwords to store.
- Refused: local and service account names (`root`, `ubuntu`, `syslog`,
  `_apt`, `systemd-*`, `slurm`, `munge`, `sssd`, `glauth`, `spack`,
  `slurmcheck`, …),
  duplicate names or uids, anything in `ssh_keys` that is not exactly one
  public key, unknown fields.
- A missing file means nobody but the admin.

Apply it to a running cluster with `pkgx task users` (see
[Operations](../operations)).

### Operators

An operator manages jobs and workers without being root: on the
frontend, `sudo slurmop` and nothing else.

| Command | Does |
|---|---|
| `sudo slurmop cancel JOBID…` | cancel people's jobs |
| `sudo slurmop cancel --user NAME` | cancel all of one person's jobs |
| `sudo slurmop hold JOBID…` / `release JOBID…` | keep pending jobs from starting, then let them |
| `sudo slurmop requeue JOBID…` | put running jobs back in the queue |
| `sudo slurmop drain NODE REASON…` | let a worker finish its jobs and take no more |
| `sudo slurmop resume NODE` | put a worker back in service |

slurmop only touches the jobs of directory users (uids 20000–29999), never
the admin's or a service account's; a node must be one Slurm knows, and a
reason is a short line without `=`. It cannot change a partition, a limit
or Slurm's configuration, and gives no access to anyone's files. Each use
is logged under the operator's name, by sudo and by slurmop (authpriv), and
a drain's reason starts with it (`kielbasi: disque plein`).

Removing the role takes effect at once, even in an open session: slurmop
asks the directory at each use. A new operator must log in again, since
sudo reads the groups of the login session. The admin may run slurmop too
(members of the local `sudo` group are let through: they are root already).
Operators are the directory group `hpcoperators` (gid 20001;
`slurm_operators_group` and `slurm_operators_gid` in
`ansible/vars/slurm.yml`), which sudo lets run `/usr/local/sbin/slurmop`.

Why a tool and not sudo rules over `scontrol`: a `*` in sudoers matches
spaces too, so a rule for `scontrol update NodeName=* State=RESUME` would
also allow `… PartitionName=batch MaxTime=INFINITE`, run as root, whom no
limit binds.

## Job limits

In `ansible/vars/slurm.yml`, read by both `site.yml` and `verify.yml`:

| Variable | Default | Meaning |
|---|---|---|
| `slurm_partition_name` | `batch` | the partition users submit to (the default one) |
| `slurm_max_time` | `00:05:00` | longest a job may run, `[days-]hh:mm:ss` or `INFINITE`; also the time a job gets when it names no `--time` |
| `slurm_max_nodes` | `2` | most nodes one job may ask for, or `UNLIMITED`; several jobs still share the workers |
| `slurm_verify_partition` | `verify` | `verify`'s partition: every worker, a day at most |
| `slurm_verify_groups` | `[sudo, slurmcheck]` | the only groups that may submit to it or see it |

With `EnforcePartLimits=ALL`, `sbatch`, `srun` and `salloc` refuse a job
that asks for more at once, rather than leaving it pending forever. The
limits bind every user but root. `verify` checks them: a job within them is
accepted, one node more and a day are refused. Change them and run
`pkgx task provision`: no need to rebuild the cluster.

## Role variables

Defaults in `ansible/roles/*/defaults/main.yml`:

| Variable | Default | Role |
|---|---|---|
| `slurm_ram_fraction` | `0.95` | slurmctld — share of each worker's RAM given as `RealMemory` |
| `nfs_export_path` | `/srv/home` | nfs_server, nfs_client |
| `nfs_mount_path` | `/home` | nfs_client |
| `common_system_accounts` | `[munge, slurm]` | common |
| `glauth_version`, `glauth_sha256` | `2.5.4`, its release digest | glauth |
| `glauth_base_dn` | `dc=slurmstack` | glauth, sssd |
| `glauth_port` | `636` | glauth, sssd |
| `apps_frontend_packages`, `apps_worker_packages` | gcc, g++, gfortran, Lmod; on the frontend also cmake and Spack's prerequisites | apps |
| `spack_version`, `spack_sha256` | `1.2.2`, its release digest | spack |
| `spack_packages_commit` | a commit of `spack-packages` `releases/v2026.06` | spack — the recipes, and with them the checksum of every source |
| `spack_target` | `x86_64_v3` | spack — every build runs on frontend and workers alike |
| `spack_specs` | `openmpi@5.0.10 +internal-pmix fabrics=none schedulers=slurm`, `gmsh@4.15.1 ~fltk ~opencascade ~med ~cgns ~mpi +openmp …` | spack — installed site-wide on provisioning, as the `spack` account |
| `spack_install_timeout` | `10800` | spack — seconds a background `spack install` may take |
| `spack_build_stage` | `/var/lib/spack-stage` | spack — private to the `spack` account, never the shared `/tmp` |

Role variables are defaults inside the embedded playbooks: change them in
`ansible/roles/*/defaults/main.yml` and rebuild (`pkgx task build`).

slurmstack itself passes `slurm_cluster_name`, `slurm_private_cidr`,
`slurm_frontend_ip`, `slurm_nfs_ip`, `nfs_volume_device`, `munge_key`,
`ldap_users`, `ldap_ca_pem`, `ldaps_cert_pem`, `ldaps_key_pem`,
`glauth_sssd_password` and `glauth_sssd_bcrypt_hex`.

## Command

```text
slurmstack <command> [flags]

  up          apply, then provision, then verify
  plan        terraform init + plan, changing nothing
  apply       terraform init + apply
  provision   pin host keys from the consoles, then run ansible/site.yml
  verify      run ansible/verify.yml
  users       apply users.yaml to a provisioned cluster: homes, glauth, SSSD
  ssh-config  print an OpenSSH config reaching every node (-o FILE to write it)
  destroy     terraform destroy (needs --yes)
```

The command line is built with [cobra](https://github.com/spf13/cobra): flags
are POSIX (`--openrc FILE`), and `slurmstack <command> --help` lists them.
**Compatibility:** the single-dash spellings of earlier versions (`-openrc
FILE`, `-var-file=x`, `-yes`) are still accepted and mean the same thing, so
existing Taskfiles keep working. `-o` and `-h` are the short forms of
`--output` and `--help`.

| Flag | Default | Meaning |
|---|---|---|
| `--openrc` | `$SLURMSTACK_OPENRC` | openrc file to read the `OS_*` variables from; empty uses the environment. |
| `--workdir` | `.slurmstack` | State: Terraform working copy and state, `known_hosts`, `munge.key`. Mode 0700. |
| `--var-file` | `terraform.tfvars` | Terraform variables file. |
| `--terraform` | first of `terraform`, `tofu` on the PATH | Terraform binary. |
| `--identity` | ssh-agent | Private key file (must have no passphrase; otherwise use the agent). |
| `--users` | `users.yaml` | The people glauth publishes. |
| `--yes` | off | `destroy` only: really destroy every resource. |
| `--output`, `-o` | stdout | `ssh-config` only: write the config to this file (mode 0600). |

## Tasks

`pkgx task --list` shows them: `build`, `plan`, `deploy`, `provision`,
`users`, `verify`, `ssh-config`, `teardown`, `test`, `lint`, `docs`,
`landing`. Pass
`OPENRC=path` on the line instead of exporting `SLURMSTACK_OPENRC`.
