---
title: Architecture
weight: 3
---

## Infrastructure (Terraform)

| File | Creates |
|---|---|
| `network.tf` | private network and subnet, router with its gateway on the external network, router interface |
| `secgroups.tf` | `<cluster>-internal`: all TCP/UDP/ICMP between members, egress anywhere; `<cluster>-ssh`: port 22 from `allowed_ssh_cidrs`, frontend only. Both drop OpenStack's default rules. |
| `compute.tf` | keypair, one port per server (carrying its security groups), the frontend and the workers, the floating IP and its association to the frontend's port |
| `storage.tf` | the NFS server and its port, the `/home` volume and the `/opt/apps` volume (the shared Spack instance) with their attachments |
| `outputs.tf` | what slurmstack reads back: ids, names and private addresses of every server, the floating IP, the subnet CIDR, the volume id |

Ports are created explicitly so no server ever lands on the project's
`default` security group.

## Configuration (go-ansible)

`ansible/site.yml`, run in process:

1. **All nodes** — wait for SSH and for cloud-init's `boot-finished`, gather facts.
2. **common**, then **admin_keys** (all) — `/etc/hosts` with every node; the admin's key (Terraform's keypair) copied once to the root-owned `/etc/ssh/authorized_keys/<user>`, `users.yaml`'s `admin_ssh_keys` rewritten on every run into `/etc/ssh/authorized_keys/<user>.extra`, and sshd told to read both first (so neither depends on the NFS share); `munge` and `slurm` system accounts created before any package does, then an assertion that their uid/gid are the same everywhere; `nfs-common`.
3. **nfs_server** (nfs) — ext4 on the volumes (by their `/dev/disk/by-id/virtio-…` serial, stable where `/dev/vdb` is not), mounted on `/srv/home` and `/srv/apps`; a home for the admin user and one per directory user (numeric owner, mode 0700); `/srv/home` exported to the private subnet with `root_squash`; `/srv/apps` exported read-write to the frontend only and read-only to the rest of the subnet, its Spack tree owned by the `spack` account.
4. **glauth** (frontend) — the pinned GLAuth release, checked against its sha256 before it is put on the PATH; LDAPS only, on the frontend's private address, with a certificate from the cluster CA; a hardened unit (`DynamicUser`, configuration and key passed as systemd credentials, `IPAddressAllow` the private subnet). It publishes the people of `users.yaml` as `posixAccount` entries with their SSH keys, the groups `hpcusers` (everyone) and `hpcoperators` (those with `role: operator`), and holds one account with a password: SSSD's, read-only.
5. **sssd** (frontend and workers; not the NFS server) — SSSD over LDAPS, trusting only the cluster CA, gives NSS (users and groups), PAM access (members of `hpcusers`) and SSH keys (`AuthorizedKeysCommand sss_ssh_authorizedkeys`; directory users have no `authorized_keys` file). The play refuses a directory name that is a local account of the node, and checks every directory user and `hpcusers` resolve.
6. **apps** (frontend and workers) — gcc 13 (with g++ and gfortran) and Lmod from the Ubuntu archive, plus Spack's prerequisites and cmake (a build tool Spack uses) on the frontend; the shared apps tree mounted at `/opt/apps`, read-write on the frontend and read-only on the workers, `nosuid,nodev`.
7. **spack** (frontend) — the pinned Spack release, checked against its sha256, unpacked into `/opt/apps/spack` as the `spack` account (root is squashed on that export, and no build ever runs as root; builds are staged in the private `/var/lib/spack-stage`, never the shared `/tmp`); site configuration: install tree outside the release directory, package recipes pinned to a commit, Ubuntu's gcc, cmake, Slurm, OpenSSH and Perl as externals, every build targeting `x86_64_v3`, Lmod modules in `/opt/apps/modules/spack` (names without hash, implicit dependencies hidden), which `/etc/lmod/modulespath` points every node at. It builds `spack_specs`, in the background and polled: **OpenMPI 5.0.10** with its bundled PMIx 5 (Ubuntu's Slurm speaks PMIx 5, so `srun --mpi=pmix` launches it), `schedulers=slurm`, TCP and shared memory; **gmsh 4.15.1** headless (no FLTK GUI, no OpenCASCADE, no MED/CGNS), with its CLI, `libgmsh` and Python API.
8. **Slurm** (frontend and workers, one play) — each node records its CPUs and memory; the shared munge key; `/srv/home` mounted as `/home` with `nosuid,nodev`; on the frontend, `slurmctld` with `slurm.conf` (`SlurmctldParameters=enable_configless`, one `NodeName` per worker from the values each recorded; the users' partition with its job limits, `EnforcePartLimits=ALL`, and `verify`'s own partition, open only to the groups of `slurm_verify_groups`; see [Configuration](../configuration#job-limits)), `cgroup.conf`, and `mpi.conf` (`PMIxEnv=PMIX_MCA_psec=native`: Ubuntu's Slurm offers PMIx munge security first, which the PMIx inside Spack's OpenMPI lacks, so without it every rank prints a warning before falling back); a change to any of them is pushed to the workers with `scontrol reconfigure`.
9. **Workers** — `slurmd`, and `slurm-client` for the `srun` a batch script calls on the first worker of its allocation; `slurmd` runs with `SLURMD_OPTIONS="--conf-server <frontend>"`, and a drop-in clearing the packaged unit's `ConditionPathExists=/etc/slurm/slurm.conf`, which would otherwise keep a configless slurmd from starting.
10. **Operators** (frontend) — `slurmop`, built for Linux by `task build` and carried inside slurmstack, staged in root's `/etc/slurmstack`, then installed in `/usr/local/sbin` by a rename (go-ansible's `copy` writes in place); the name of the operators' group, which slurmop reads to ask the directory at each use; the sudo rule letting `hpcoperators` run it and nothing else, through go-ansible's `sudoers` module (temporary file, `visudo -c`, rename). See [Configuration](../configuration#operators).
11. **Message of the day** (frontend and workers) — `/etc/motd` from `users.yaml`'s `motd` and a line of job limits; Ubuntu's `/etc/update-motd.d` scripts lose their execute bit, and those that are links into a package (`50-landscape-sysinfo`) are removed.

`ansible/verify.yml` then waits for every worker to be idle, runs `srun` on
all of them, checks gcc and gfortran on each, that `/opt/apps` is mounted
read-only, `nosuid,nodev` and owned by `spack`; then, as `slurmcheck` (a
local account with no sudo and no login, so nothing from `/opt/apps` runs
as the admin), compiles an MPI program with Spack's `mpicc`, runs one rank
per worker with `srun --mpi=pmix`, does it again as an `sbatch` job (a
login shell on the first worker that loads the `openmpi` module and calls
`srun` there), runs gmsh on every worker, and checks
Lmod finds `openmpi` and `gmsh` there; these jobs span every worker, so
they run in `verify`'s partition. Then, in the users' partition, a job
within the limits is accepted and one asking a node too many or a day is
refused (`sbatch --test-only`, nothing runs). On the frontend, sudo
allows each operator exactly `slurmop` and every other directory user
nothing, `hpcoperators` holds exactly the operators, slurmop refuses the
admin's jobs, an unknown node and a reason smuggling a field, and the
admin's extra keys are exactly `admin_ssh_keys`, `/etc/motd` holds the
message and the limits, and no login script of Ubuntu's can run. Last, every directory user has the
directory's uid on each worker, and `/home` is a mount point on each.

`ansible/users.yml`, run by `pkgx task users`, applies `users.yaml` to a
running cluster: the `homes` role on the NFS server, `glauth` on the
frontend, `sssd` on the Slurm nodes.

## The driver (Go)

| Package | Does |
|---|---|
| `assets.go` | embeds `terraform/` and `ansible/`; extracted into the state directory on every run |
| `internal/openrc` | parses an openrc file without a shell |
| `internal/cloud` | authenticates with gophercloud and reads console logs |
| `internal/hostkeys` | extracts the host keys cloud-init prints, formats `known_hosts` lines |
| `internal/cluster` | turns `terraform output` into a validated description |
| `internal/mungekey`, `internal/secret` | create the munge key and SSSD's bind password once (0600), refuse one readable by others; keep a stable bcrypt hash for glauth |
| `internal/users` | reads `users.yaml`: names, explicit uids in 20000–29999, SSH keys; refuses reserved names, duplicates, anything that is not one public key |
| `internal/pki` | the cluster CA (ECDSA P-256) and glauth's LDAPS certificate for the frontend's private address and name, re-issued only when the host changes or it nears expiry |
| `internal/provision` | inventory, extra vars, the SSH connector (jump through the frontend, retries while servers boot), the playbook run |
| `cmd/slurmstack` | the commands, Terraform through `terraform-exec` |

## Why go-ansible needs help here

go-ansible's `DefaultConnect` does not read `ansible_ssh_common_args`, so it
cannot use `-o ProxyJump`. slurmstack replaces the engine's connector:
hosts carrying the `slurmstack_jump` variable are dialled through one shared
SSH connection to the frontend. `ProxyJump` is proposed upstream in
[go-remoteexec/transport#23](https://github.com/go-remoteexec/transport/pull/23)
and [go-ansible/playbook#189](https://github.com/go-ansible/playbook/pull/189).

go-remoteexec/transport v0.5.0 runs `Put` without `become`, so every write to
a root-owned file failed. `third_party/transport` carries the fix
([go-remoteexec/transport#24](https://github.com/go-remoteexec/transport/pull/24))
until it is released; see `third_party/README.md`.

The playbooks also work around these go-ansible behaviours, each found on a
live run and commented where it is worked around:

| go-ansible behaviour | Worked around by |
|---|---|
| `copy` ignores `owner`, `group` and `force` | a `file` task after the copy ([modules#106](https://github.com/go-ansible/modules/pull/106)) |
| `user` takes no `uid` or `group` | creating the system accounts first, then asserting their ids match ([modules#105](https://github.com/go-ansible/modules/pull/105)) |
| `getent` puts its result in the registered variable, not in `ansible_facts` ([modules#109](https://github.com/go-ansible/modules/issues/109)) | reading whichever is there |
| no fact or `set_fact` value reaches a later play ([playbook#193](https://github.com/go-ansible/playbook/issues/193)) | the frontend and the workers share the Slurm play, and record CPUs and memory with `set_fact` |
| `pre_tasks:` run after the roles ([playbook#190](https://github.com/go-ansible/playbook/issues/190)); `when:` and `tags:` on an entry of `roles:` are ignored ([playbook#192](https://github.com/go-ansible/playbook/issues/192)) | plain `tasks:` with `import_role` / `include_role` |
| hosts that skip a conditional block-type task (`include_role`, `block`, …) drop out of the rest of the play ([playbook#191](https://github.com/go-ansible/playbook/issues/191)) | slurmd in a play of its own |
| `systemd_service` ignores `daemon_reload` ([modules#107](https://github.com/go-ansible/modules/issues/107)) and wants a `name` for a reload alone ([modules#108](https://github.com/go-ansible/modules/issues/108)) | `systemctl daemon-reload` as its own task, when the drop-in changed |
| `failed_when` / `changed_when` see the result only as `result`, not under its `register:` name ([playbook#194](https://github.com/go-ansible/playbook/issues/194)) | an `assert` task after it |
| `wait_for_connection` cannot wait, since the engine connects before the task ([playbook#195](https://github.com/go-ansible/playbook/issues/195)) | the connector retries network errors for 3 minutes |
| `file` (and `lineinfile`, `blockinfile`, `replace`, `stat`) know no argument aliases, such as `dest` for `path` ([modules#110](https://github.com/go-ansible/modules/issues/110), fix proposed in [modules#111](https://github.com/go-ansible/modules/pull/111)) | `path:` |
| `assert` checks `that` without the loop's variables, so `item` is undefined there ([playbook#196](https://github.com/go-ansible/playbook/issues/196), fix proposed in [playbook#197](https://github.com/go-ansible/playbook/pull/197)) | the results are joined into two strings with `set_fact`, compared by one `assert` |
