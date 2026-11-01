---
title: Operations
weight: 5
---

## Reach the nodes

```sh
pkgx task ssh-config          # writes .slurmstack/ssh_config
ssh -F .slurmstack/ssh_config slurm-frontend
ssh -F .slurmstack/ssh_config slurm-worker-01   # through the frontend
```

The config pins every host to `.slurmstack/known_hosts` with
`StrictHostKeyChecking yes`.

## Change the cluster

Edit `terraform.tfvars`, then `pkgx task plan` and `pkgx task deploy`.
`deploy` is safe to re-run: Terraform changes only what differs, and the
playbooks converge. Adding workers re-renders `slurm.conf` and restarts
slurmctld.

## Add or remove a user

Edit `users.yaml` (add a person, change or add a key, remove someone and
move their uid to `retired_uids`), then:

```sh
pkgx task users
```

It applies the list to the running cluster without replaying everything:
it creates the missing homes on the NFS server, rewrites glauth's list
(restarting glauth only if it changed), empties SSSD's cache on the frontend
and every worker so the change takes effect at once, refuses a name that is
a local account of a node, and checks every user resolves everywhere. The
host keys pinned by `provision` are reused. A removed user's home stays on
the NFS server: delete it by hand once its files are no longer needed.

The same command makes someone an operator or no longer (`role:
operator`; a new operator logs in again, a removed one loses slurmop at
once), and rewrites the admin's extra keys (`admin_ssh_keys`) on every
node: a key removed from the list stops working at once. It also rewrites
the message of the day (`motd`) on the frontend and the workers.

## Operate the queue

An operator (see [Configuration](../configuration#operators)), logged in
on the frontend:

```sh
squeue                                        # everyone's jobs
sudo slurmop cancel 1234                      # one job
sudo slurmop cancel --user alice              # all of alice's jobs
sudo slurmop drain slurm-worker-03 "disque plein"
sinfo                                         # slurm-worker-03 drng, then drain
sudo slurmop resume slurm-worker-03
```

What operators did is in the frontend's journal:
`journalctl -t sudo -t slurmop` (as the admin).

## Install software for everyone

The site Spack lives in `/opt/apps/spack` and builds as the `spack`
account, never as root. On the frontend:

```sh
sudo -u spack -H env -C /opt/apps /opt/apps/spack/current/bin/spack install zlib-ng
sudo -u spack -H env -C /opt/apps /opt/apps/spack/current/bin/spack module lmod refresh --delete-tree -y
```

`env -C /opt/apps` matters: Spack fails ("Permission denied: '/home/ubuntu'")
when started from a directory the `spack` account cannot enter. To have
`provision` install a list of specs every time, set `spack_specs` in
`ansible/roles/spack/defaults/main.yml` and rebuild (`pkgx task build`): the
playbooks are embedded in the binary.
Everything lands on the shared volume: the workers see it read-only, and
users find it with `module avail`. Ubuntu's gcc 13 and cmake are declared
as externals, so Spack builds with them rather than again. OpenMPI and gmsh
are installed this way on every `provision` (`spack_specs`); a first build
takes about half an hour on a vd.16 frontend.

Users can chain their own Spack onto the site one, to build in their home
what the site does not provide:

```yaml
# ~/.spack/upstreams.yaml
upstreams:
  site:
    install_tree: /opt/apps/spack/opt
```

## Re-run the configuration only

```sh
pkgx task provision
pkgx task verify
```

## State directory

`.slurmstack/` (mode 0700) holds:

| Path | What |
|---|---|
| `terraform/` | the extracted configuration, `.terraform/` and `terraform.tfstate` |
| `ansible/` | the extracted playbooks |
| `known_hosts` | host keys read from the consoles (0600) |
| `munge.key` | the cluster's munge key (0600) |
| `pki/ca.crt`, `pki/ca.key` | the cluster CA SSSD trusts for LDAPS (key 0600) |
| `pki/ldaps.crt`, `pki/ldaps.key` | glauth's certificate for the frontend's private address (key 0600) |
| `sssd.password`, `sssd.bcrypt` | SSSD's bind password and the bcrypt hash glauth checks (0600) |
| `ssh_config` | written by `task ssh-config` |

On the cluster, `/opt/apps` (the `apps` volume on the NFS server) holds
`spack/releases/` (the checked tarball and the unpacked release),
`spack/current` (a link to it), `spack/opt/` (what Spack built),
`spack/spack-packages/` (the pinned recipes), `spack/cache/`, `spack/home/`
(the `spack` account's home) and `modules/spack/` (the Lmod tree). Builds
are staged in `/var/lib/spack-stage` on the frontend. `destroy` deletes the
volume too.

⛔ Losing `terraform.tfstate` orphans the resources: Terraform no longer
knows them and `destroy` cannot remove them.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `the ssh-agent holds no key` | run `ssh-add` with the key matching `ssh_public_key_path` |
| `sets no OS_PASSWORD and stdin is not a terminal` | the openrc prompts for the password; run from a terminal or export `OS_PASSWORD` |
| `neither terraform nor tofu is on the PATH` | run through `pkgx +terraform.io --`, as the Taskfile does |
| a node `UNREACHABLE` after 3 minutes | it never answered on port 22: check its console log in Horizon |
| `knownhosts: key mismatch` | the server was rebuilt outside slurmstack: run `provision` again to re-read its console |
