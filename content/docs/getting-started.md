---
title: Getting started
weight: 1
---

## What you need

- [pkgx](https://pkgx.sh): every tool — Go, Terraform, Hugo, go-task — is run through it, nothing is installed system-wide.
- The **openrc file** of your OpenStack project (Horizon → *API Access* → *Download OpenStack RC File*).
- An SSH key loaded in your **ssh-agent**. A key with a passphrase must be added first:

  ```sh
  ssh-add ~/.ssh/id_ed25519
  ```

  slurmstack checks this before connecting and says so if the agent holds no key.

## Configure

Copy the example variables and say who may reach the frontend over SSH —
there is no default on purpose:

```sh
cp terraform/terraform.tfvars.example terraform.tfvars
```

```hcl
allowed_ssh_cidrs = ["129.104.0.0/16"]
```

Every other variable has a default; see [Configuration](../configuration).
The default workers are `vd.32` (32 vCPUs, 64 GB each): check the project's
quota covers four of them, or set a smaller `worker_flavor`.

List the cluster's people, who will log in by SSH key:

```sh
cp users.yaml.example users.yaml
```

## Deploy

```sh
export SLURMSTACK_OPENRC=~/Downloads/my-project-openrc.sh
pkgx task plan      # what would be created, changing nothing
pkgx task deploy    # apply, provision, verify
```

`deploy` runs three steps, each also available on its own:

1. **apply** — `terraform init` and `apply`: network, subnet, router, security groups, keypair, ports, the frontend, the NFS server, the workers, the `/home` volume, the floating IP.
2. **provision** — reads each server's SSH host keys from its console, writes `.slurmstack/known_hosts`, prepares the cluster CA and glauth's certificate, then runs `ansible/site.yml` with go-ansible's engine: NFS, glauth, SSSD, munge, Slurm.
3. **verify** — runs `ansible/verify.yml`: waits for every worker to be idle, runs `srun` on all of them, runs an MPI program with `srun` and again as an `sbatch` job, checks every user of `users.yaml` has its uid on each, and that `/home` is the NFS mount on each.

If the openrc file does not contain the password, slurmstack asks for it on
the terminal.

## Use the cluster

```sh
pkgx task ssh-config
ssh -F .slurmstack/ssh_config slurm-frontend            # the admin
ssh -F .slurmstack/ssh_config -l alice slurm-frontend   # a user from users.yaml
sinfo
srun --nodes=2 hostname
module avail                                            # openmpi, gmsh
module load openmpi gmsh
mpicc -o hello hello.c
srun --mpi=pmix --nodes=2 --ntasks-per-node=2 ./hello   # two ranks per node
gmsh -version
```

A job may run 5 minutes and use 2 nodes at most; one that asks for more is
refused at submission (see [Configuration](../configuration#job-limits)).

gcc 13 (C, C++, Fortran) comes from Ubuntu on the frontend and every
worker. OpenMPI 5 and gmsh are built by the site Spack and loaded with
`module load`, in a job as on the frontend. MPI programs are launched by
Slurm with `--mpi=pmix` (or with `mpirun` inside an `salloc`). gmsh is
headless: its command line, `libgmsh` and the Python API (`import gmsh`
once the module is loaded); no GUI. See [Operations](../operations) to add
software.

A batch job, `hello.sbatch` (C++ here, built with `mpicxx -O2 -o hello
hello.cpp` once `openmpi` is loaded):

```bash
#!/bin/bash -l
#SBATCH --job-name=hello-mpi
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=4
#SBATCH --mem-per-cpu=1G
#SBATCH --time=00:05:00
#SBATCH --output=hello-%j.out

module purge
module load openmpi

srun --mpi=pmix ./hello
```

`sbatch hello.sbatch`, then `squeue -u $USER`. The script runs on the first
worker of the allocation: `bash -l` gives it Lmod's `module`, and `srun`
starts the ranks on every node allocated. Slurm hands out memory as well as
cores (`CR_Core_Memory`), so ask for it: a job without `--mem-per-cpu` or
`--mem` holds a whole node's memory. Accounting is off, so `sacct` has
nothing to show.

## Tear down

```sh
pkgx task teardown
```

It asks for confirmation: the volume holding `/home` is destroyed with the
servers.
