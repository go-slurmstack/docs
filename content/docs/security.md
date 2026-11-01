---
title: Security
weight: 4
---

What slurmstack protects, how, and — stated as plainly — what it does not.
Four audits of the code (2026-10-12/13): the first, of the cluster, found
nothing critical or high; the second, of the glauth/SSSD directory, and the
third, of the shared Spack tree, each found one high issue, fixed before it
shipped; the fourth, of OpenMPI and gmsh, found nothing high. Findings and
outcomes are at the end.

## The trust model

**The cluster is one trust domain.** Root on any node can become any user on
every node:

- every Slurm node holds the same munge key, so root on one can forge a
  credential for any uid to the others;
- `/home` is NFS with `sec=sys` (`AUTH_UNIX`): root on a client can write in
  any user's home as that user, including their `~/.ssh/authorized_keys`.

That is the usual shape of a small Slurm cluster, and it is fine for
teaching. Do not give students root on the nodes. If workers must not be
trusted, use Kerberos (`sec=krb5p`) and `pam_slurm_adopt`, which are out of
scope.

What is kept out of that domain: the **admin's** SSH key. It is read from
each node's own disk before `/home` is mounted, kept in the root-owned
`/etc/ssh/authorized_keys/<user>`, and sshd is told to read it from there
first. So root on a worker cannot reach the admin's `sudo` on the frontend or
the NFS server by editing a file on the share.

## The directory

- **glauth** listens for LDAPS only, on the frontend's private address.
  Port 636 is not in the frontend's public security group, and the unit's
  `IPAddressAllow` limits it to the private subnet as a second fence (the
  floating IP is 1:1 NAT onto that address, so binding to it alone would
  not hide the port).
- **TLS** comes from the cluster's own CA (ECDSA P-256), generated in Go in
  `.slurmstack/pki`. Its key never leaves your machine. SSSD trusts that CA
  alone (`ldap_tls_reqcert = demand`).
- **No user has a password.** People log in by SSH key, which sshd gets
  from SSSD (`sss_ssh_authorizedkeys`); `auth_provider = none` means no
  password is ever checked against LDAP. The one password in the directory
  is SSSD's read-only bind account: random, kept 0600 in the state
  directory, stored in glauth only as a bcrypt hash, and on the nodes in a
  root-only `sssd.conf`. Anonymous search is refused. There is no
  failed-bind limit on purpose: glauth counts per source address, so any
  user on a node could lock that node's SSSD out, and a 288-bit random
  password needs no such limit.
- **Keys are re-encoded**: only the key itself, never its comment or
  options as pasted, reaches the playbooks. go-ansible re-renders extra
  vars as templates, so a comment like `{{ lookup('pipe', …) }}` would
  otherwise run on the machine that deploys, or publish a secret.
- **Access**: SSSD lets in members of `hpcusers` only, and takes ids only
  from the people's range (20000–29999). Directory users get their SSH keys
  from the directory alone (`AuthorizedKeysFile none` for `hpcusers`), so a
  key they add to their home does not outlive their removal. Names that are
  local accounts are refused (`users.yaml` check, and each node's own
  `/etc/passwd` checked by the play), because sshd asks SSSD for keys by
  name. The NFS server runs no SSSD: no directory user can log in there.
- **Revocation**: remove the user from `users.yaml` (and add their uid to
  `retired_uids`, so it is never given to someone else with their old
  files), then `pkgx task users`: glauth drops them and SSSD's cache is
  emptied on every Slurm node, so they are refused at once (otherwise
  within 5 minutes). SSSD is restarted for this, so names and logins fail
  for a few seconds on each node while the task runs.
  Sessions and jobs already running are not stopped: `pkill -u <name>` and
  `scancel -u <name>` on the frontend.
- **The release** is pinned and its sha256 is checked before the binary is
  put on the PATH. GLAuth publishes no checksum file, signature or
  attestation, so that digest is GitHub's own for the asset: it proves the
  bytes are the ones GitHub served for that release, not who built them.
- glauth runs as a dynamic user with no capability but
  `CAP_NET_BIND_SERVICE`; its configuration and key reach it as systemd
  credentials, root-only on disk.

## Software

- Ubuntu's gcc, cmake and Lmod come from the archive, signed by Ubuntu.
  OpenMPI and gmsh are built by Spack, under the rules below.
- `verify` runs what Spack built only as `slurmcheck`, a local account
  with no sudo and no login; never as the admin.
- **Spack** is the pinned release; its tarball's only integrity anchor is
  GitHub's asset digest, checked with `sha256sum` before it is unpacked:
  Spack publishes no checksum file or signature.
- Spack checks every source it downloads against the sha256 in its package
  recipes, over TLS (`checksum: true`, `verify_ssl: true`). The recipes are
  pinned to one commit of `spack-packages`: that commit is the trust anchor
  for everything Spack builds.
- Builds run as the `spack` account, never as root, from the frontend only,
  in a private stage (`/var/lib/spack-stage`, root-created, `spack`-owned,
  0700), never in the shared `/tmp`, where another user could swap a source
  after its checksum was checked.
- The tree is owned by `spack`; the workers mount it read-only, every node
  with `nosuid,nodev`; users cannot write it. The frontend's read-write
  grant is by its private IP, which Neutron's port security (on, and never
  turned off here) keeps from being spoofed.
- **The `spack` account is a trust anchor for the whole cluster**, like the
  `spack-packages` commit: whoever controls it changes the code every user
  runs when they `module load`, or chain their own Spack onto the site one.
  Treat it like root: the frontend's admin (`sudo -u spack`) controls it,
  and squashing root on the export does not change that. The admin should
  not load site modules from an account with `sudo`. `verify` runs code
  from `/opt/apps` (`mpicc`, the MPI test, gmsh, the modulefiles) only as
  `slurmcheck`, and `spack location` only as `spack`; as the admin it only
  checks the tree's owner and mount options. Its MPI and gmsh checks rest
  on code `spack` controls, so a compromised `spack` could make them pass.
- **Service accounts cannot log in**: `slurmcheck`, `spack`, `munge` and
  `slurm` are in sshd's `DenyUsers`, so a key planted in their home opens
  neither a shell nor a tunnel.
- OpenMPI is built with its default `+rsh` launcher: a user with an SSH key
  can start ranks on the workers over SSH, outside Slurm's cgroups. Like
  the missing `pam_slurm_adopt` (L8), this is accepted for a teaching
  cluster; building OpenMPI `~rsh` would remove it.

## The network

- Only the frontend has a public address, and port 22 is open only to
  `allowed_ssh_cidrs`, which has no default.
- Both security groups drop OpenStack's default rules. Inside the cluster,
  ingress is allowed only from members of `<cluster>-internal`, which also
  narrows the NFS export (`/24`) to cluster members.
- Every server has an explicit port, so none falls on the project's
  `default` security group.
- Egress is open, for apt and DNS. Restricting it to 53/80/443/123 is
  possible, but not done.

## SSH

- **Host keys are read from each server's console** through the OpenStack API
  before the first connection. The first key block cloud-init prints is
  pinned in `.slurmstack/known_hosts`, the only file the connector checks.
  Host key checking is never turned off and nothing is trusted on first use.
- `OS_AUTH_URL` and the compute endpoint must be `https`: the console log is
  only as trustworthy as the channel it arrives on.
- Who can write to the console: the guest itself, and project members who
  type at a serial console (it is echoed into the log). The *first* block is
  kept because cloud-init prints it once, early in the first boot; a later
  one does not replace it.
- Workers and the NFS server are reached through the frontend; no agent
  forwarding.
- The admin account opens with Terraform's key (`/etc/ssh/authorized_keys/
  <admin>`, copied once) and `users.yaml`'s `admin_ssh_keys`
  (`<admin>.extra`, rewritten on every run). Both are root-owned, outside
  the share. Each of these keys is root on every node, and all log in as
  the same account: prefer a nominal account, an operator if needed.

## Message of the day

- `users.yaml`'s `motd` reaches every login screen, so it is checked
  before anything runs: no template delimiters (go-ansible re-renders
  extra vars, and `{{ munge_key }}` would print the key to every user), no
  control character (an escape sequence would drive users' terminals), at
  most 4000 bytes.
- Ubuntu's login scripts are disabled rather than trusted: they fetch
  Canonical's news over the network at login (`50-motd-news`) and print
  system details to anyone who logs in.

## Operators

- An operator (`role: operator`) may run `sudo /usr/local/sbin/slurmop` on
  the frontend, and nothing else. slurmop is a small Go program: every
  argument is checked whole (job ids, user and node names cannot start
  with `-`; a drain reason holds no `=` nor control byte), a job or a
  user must belong to the directory's range (20000–29999), so the admin's,
  `slurmcheck`'s and the service accounts' jobs are out of reach, and it
  runs `scancel`/`scontrol` from absolute paths, without a shell, with a
  fixed environment.
- sudo admits by the groups of the login session, fixed at login. slurmop
  therefore asks the directory again at each use (`getent group`, after
  `task users` emptied SSSD's cache): someone made no longer an operator
  is refused at once, even in a session opened before. A new operator
  needs a new login for sudo to admit them.
- Each use is logged twice under the operator's name: by sudo and by
  slurmop (authpriv, with the outcome); a drain's reason starts with it.
- The rule reaches `/etc/sudoers.d` only after `visudo -c` accepted it
  (go-ansible's `sudoers` module: temporary file, check, rename). The
  binary is installed by a rename too, never rewritten in place.
- By design, an operator can release a hold the admin set, resume a node
  the admin drained or Slurm marked down, and drain every worker (which
  also stops `verify`). These are an operator's job.

## Secrets

| Secret | Where | Protection |
|---|---|---|
| OpenStack password | openrc file, or typed at a prompt | parsed, never sourced; only `OS_*` variables are taken; passed to terraform through the environment, never argv; never logged; an unparsable line is reported by number and first word only |
| cluster CA and LDAPS keys | `.slurmstack/pki/*.key` | ECDSA P-256, created `0600`, refused if readable by others; the CA key never leaves your machine; glauth's key is `0600` on the frontend and handed to glauth as a systemd credential |
| SSSD bind password | `.slurmstack/sssd.password` | random, `0600`; glauth holds only its bcrypt hash; on the nodes in root-only `sssd.conf`; every task writing it is `no_log` |
| munge key | `.slurmstack/munge.key` | 96 random bytes, created `0600` with `O_EXCL`, refused if readable by others; handed to the engine in memory; `no_log` on every task; `0400 munge:munge` on the nodes |
| terraform state | `.slurmstack/terraform/` | made `0600` after every apply and destroy; holds no credentials (the provider block is empty, the keypair is public only) but the whole topology |
| `/home` | Cinder volume | **not encrypted at rest** on VirtualData today: its `encrypted` volume type is refused ("Invalid encryption spec"); set `nfs_volume_type` to a working encrypted type where one exists |

The state directory is created `0700`. An existing directory passed as
`-workdir` is not chmodded; slurmstack warns if others can read it.

## Supply chain

- Terraform provider pinned `~> 3.4` with a lock file carrying hashes for
  every platform; the lock file is embedded in the binary.
- The image is selected by name **and** `visibility = "public"`, so a
  project-shared image with the same name cannot be picked instead.
- Go dependencies: `govulncheck` reports nothing reachable. It lists
  GO-2026-5932 in `golang.org/x/crypto/openpgp`, which is never imported.
- CI: `contents: read`, no `pull_request_target`, no secrets,
  `persist-credentials: false`. Actions are pinned by tag; Renovate's
  `helpers:pinGitHubActionDigests` turns them into digests.
- `third_party/transport` is a copy of go-remoteexec/transport with one
  unreleased fix (see its README), to be dropped once released.

## Audit findings and their outcome

| # | Finding | Outcome |
|---|---|---|
| M1 | admin key on the NFS share: root on a worker → admin everywhere | **fixed**: key kept in root-owned `/etc/ssh/authorized_keys`; `nosuid,nodev` on the mount; trust model documented |
| M2 | `/home` volume not encrypted | **open**: tried `encrypted` as the default; VirtualData's Cinder refuses it ("Invalid encryption spec"), to be reported to its operators. `nfs_volume_type` lets a working type be chosen |
| L1 | `https` not enforced on the API | **fixed** |
| L2 | last key block on the console won | **fixed**: the first one wins |
| L3 | openrc parse error echoed the line | **fixed** |
| L4 | image chosen by name only | **fixed**: `image_visibility = "public"` |
| L5 | files dropped from a release stayed in the state directory | **fixed**: removed on extraction |
| L6 | `-workdir` chmodded any directory | **fixed** |
| L7 | terraform state `0644` | **fixed** |
| L8 | open egress, cgroups | `ConstrainDevices=yes` **added**; egress and `pam_slurm_adopt` documented, not done |
| L9 | actions pinned by tag | `persist-credentials: false` **added**; digests via Renovate |
| I1 | fact rendered into `slurm.conf` without `\| int` | **fixed** |
| I2 | go-ansible `copy` ignores `force`, `owner`, `group` | worked around with `file`; upstream fix [modules#106](https://github.com/go-ansible/modules/pull/106) |

### Directory audit

| # | Finding | Outcome |
|---|---|---|
| H1 | template syntax in an SSH key's comment runs on the deploying machine (go-ansible re-renders extra vars) or publishes the munge key through glauth | **fixed**: keys re-encoded from the parsed public key; comments and options dropped; tested |
| M1 | a removed user or rotated key keeps working (`~/.ssh/authorized_keys`, 90-minute SSSD cache) | **fixed**: `AuthorizedKeysFile none` for `hpcusers`, `entry_cache_timeout = 300`, revocation documented |
| M2 | directory users can log in to the NFS server | **fixed**: no SSSD there |
| M3 | three bad binds lock a node's SSSD out for 60 s | **fixed**: failed-bind limit removed |
| L1 | a directory name equal to a local account gets its key accepted for it | **fixed**: reserved names extended, `_`/`systemd-` refused, each node's `/etc/passwd` checked |
| L2 | a reused uid inherits an old home | **fixed**: `retired_uids` are refused |
| L3, L4 | control bytes or key options passed through | **fixed** by H1 |
| L5 | a stale temp file could hold a new key at a loose mode | **fixed**: `CreateTemp` |
| L6 | an existing `pki/` directory's mode was not checked | **fixed** |
| I | CA expiry not checked | **fixed**: refused once it would expire before a certificate it signs |
| I | glauth release integrity rests on GitHub's asset digest | **open**: GLAuth publishes no checksum file, signature or attestation |

### Software audit

| # | Finding | Outcome |
|---|---|---|
| H1 | Spack's build stage in the shared `/tmp`: a user creating `/tmp/spack` first could run code as `spack`, hence in every module | **fixed**: private stage `/var/lib/spack-stage`, root-created, `spack`-owned, 0700 |
| M1 | `verify` ran `bin/spack` from the shared tree as the admin (sudo on the workers) | **fixed**: as the admin it only checks the owner and mount options; what Spack built runs as `slurmcheck` (see below) |
| M2 | the `spack` account's place in the trust model was undocumented | **fixed**: documented as a cluster-wide trust anchor |
| L1 | `spack` was not a reserved name | **fixed** |
| L2 | the `spack` account's home was reset on every run | **fixed**: one home per system account |
| I | the tree relies on Neutron anti-spoofing for the frontend's rw grant; `task users` restarts SSSD | documented |
| I | Spack's tarball rests on GitHub's asset digest | **open**: Spack publishes no checksum file or signature |

### OpenMPI and gmsh audit

| # | Finding | Outcome |
|---|---|---|
| M1 | the docs still said `verify` runs nothing from `/opt/apps` | **fixed**: it runs Spack's builds only as `slurmcheck` |
| L1 | `slurmcheck` and `spack` accepted SSH keys from their homes (no shell, but tunnels) | **fixed**: `DenyUsers` for every service account |
| L2 | OpenMPI's `+rsh` lets a user with an SSH key start ranks outside Slurm | documented, with L8; `~rsh` is the fix if wanted |
| L3 | background build logs readable under the `spack` home (0755) | **fixed**: home 0700 |
| L4 | `verify`'s MPI and gmsh checks rest on code `spack` controls | documented (trust anchor) |

### Operators and admin keys audit

No path from operator to root was found, nor to the admin's or service
accounts' jobs.

| # | Finding | Outcome |
|---|---|---|
| M1 | a demoted operator kept `sudo slurmop` in a session opened before (sudo reads the session's groups) | **fixed**: slurmop asks the directory at each use; a new operator needs a new login (documented) |
| L1 | go-ansible's `copy` writes in place: an interrupted run could leave a truncated sudoers file, checked only as a staged copy, or half a binary | **fixed**: the `sudoers` module (check, then rename); the binary staged, then installed by a rename |
| L2 | `verify` checks that could pass vacuously | **fixed**: the rule pattern is proven on the admin, every `sudo --list` must answer, refusals are checked by message, the drain test names no real node |
| L3 | the directory's group names were valid user names | **fixed**: reserved |
| L4 | only the first line of `squeue`'s answer was checked | **fixed**: every line |
| L5 | the log did not record the outcome; Slurm records a drain as root's | **fixed**: outcome logged; the reason starts with the operator |
| L6 | slurmop is built for amd64 only | documented: VirtualData's flavors are amd64 |
| I | an operator can release the admin's holds, resume nodes, drain them all | documented, by design |

Report a vulnerability privately to the maintainers; please do not open a
public issue for it.
