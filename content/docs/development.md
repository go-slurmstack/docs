---
title: Development
weight: 6
---

How the code is checked, and what is deliberately not.

## Run the checks

```sh
go vet ./...
go test -race -count=1 ./...
go run golang.org/x/vuln/cmd/govulncheck@v1.1.4 ./...
```

`CGO_ENABLED=0` builds the binaries; the race detector needs cgo, so only the
test run sets `CGO_ENABLED=1`.

## Continuous integration

`.github/workflows/ci.yml` runs on the default branch (`master`) and on pull
requests, plus a weekly schedule for the advisory database:

| Lane | What it checks |
|---|---|
| actionlint | the workflows themselves parse |
| go (Linux, macOS) | gofmt, `go vet`, build, `go test -race` with coverage |
| govulncheck | no reachable vulnerability in the code or the standard library it was built with |
| terraform | `fmt -check`, `init -backend=false`, `validate` |

This site lives in its own repository, `go-slurmstack/docs`: its workflow
builds it on every pull request and deploys it to GitHub Pages on every push to
`main`. The slurmstack source repository is private.

## Tests

- **Unit tests with fakes.** The cloud API, the console reader, the password
  prompt and the Terraform runner are replaced by fakes; no test needs
  OpenStack credentials.
- **An in-process SSH server** (`internal/provision/bastion_test.go`) tests the
  connector for real: a pinned host key is accepted; a changed or unknown host
  key and an unauthorised client are refused at once rather than retried; the
  bastion jump works, its connection is shared, and it is re-established after
  the bastion drops its clients.
- **Fuzzing** of the three parsers of untrusted text: `openrc.Parse`,
  `users.Parse` and `hostkeys.FromConsole`. Each fuzz target asserts what an
  accepted input may contain (no value that needs a shell, no template syntax or
  shell metacharacter in a key, uids in range, never a success with no key).
  The seeds run as ordinary tests on every CI run.

## Platforms

Linux and macOS. Windows is not supported: the code refuses a secret file (CA
key, generated passwords) whose POSIX mode lets group or others read it, and
Go reports mode 0666 for every file on Windows.

## What the tests cannot reach

A real OpenStack project, real Ubuntu 24.04 nodes and the Slurm daemons. The
live deployment (`task deploy`, which ends with a job on every worker) is the
test for those.
