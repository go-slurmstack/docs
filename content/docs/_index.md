---
title: Documentation
weight: 1
---

slurmstack deploys a Slurm cluster on an OpenStack cloud — written for
[VirtualData](https://virtualdata.ijclab.in2p3.fr/) at IJCLab, usable on any
Keystone v3 cloud — from one Go binary.

```text
                    allowed_ssh_cidrs
                           │ 22/tcp
                 ┌─────────▼─────────┐
   external net  │  floating IP       │
   (public-2) ───┤  router            │
                 └─────────┬─────────┘
            private network │ 10.42.0.0/24
   ┌──────────────┬────────┴─────┬───────────────────────┐
   │ frontend     │ nfs          │ worker-01 … worker-NN │
   │ slurmctld    │ /srv/home    │ slurmd (configless)   │
   │ login node   │ /srv/apps    │ /home over NFS        │
   │ glauth LDAPS │ (Cinder vol) │ SSSD → glauth         │
   │ Spack (rw)   │              │ /opt/apps (ro), Lmod  │
   └──────────────┴──────────────┴───────────────────────┘
```

{{< cards >}}
  {{< card link="getting-started" title="Getting started" subtitle="From an openrc file to a job on every worker." >}}
  {{< card link="configuration" title="Configuration" subtitle="Every Terraform variable and command flag." >}}
  {{< card link="architecture" title="Architecture" subtitle="What runs where, and why it is built that way." >}}
  {{< card link="security" title="Security" subtitle="What is protected, how, and what is not." >}}
  {{< card link="operations" title="Operations" subtitle="SSH access, re-provisioning, teardown, troubleshooting." >}}
{{< /cards >}}
