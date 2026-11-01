---
title: slurmstack
layout: hextra-home
---

{{< hextra/hero-headline >}}
  One command,&nbsp;<br class="sm:hx-block hx-hidden" />a Slurm cluster
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  Terraform builds the network, the router, the floating IP and the servers on OpenStack;&nbsp;<br class="sm:hx-block hx-hidden" />go-ansible's engine, inside the same Go binary, configures munge, NFS and Slurm.
{{< /hextra/hero-subtitle >}}

{{< hextra/hero-button text="Get started" link="docs/getting-started" >}}

{{< hextra/feature-grid >}}
  {{< hextra/feature-card title="Host keys from the console" subtitle="Every server's SSH key is read from its console through the OpenStack API before the first connection. Nothing is trusted on first use." >}}
  {{< hextra/feature-card title="Ansible without Python" subtitle="The playbooks are embedded and run in process by go-ansible. A test fails if a task names a module the engine lacks." >}}
  {{< hextra/feature-card title="Everything is a variable" subtitle="Image, flavors, network and router names, worker count, the /home volume: Terraform variables with validation." >}}
{{< /hextra/feature-grid >}}
