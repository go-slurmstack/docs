# docs

Documentation of [slurmstack](https://go-slurmstack.github.io/), a Slurm
cluster on OpenStack from one Go binary, served at
<https://go-slurmstack.github.io/docs/>.

Built with [Hugo](https://gohugo.io) and the
[tannevaled/hextra](https://github.com/tannevaled/hextra) fork, a git
submodule in `themes/hextra` pinned to a release tag (v0.24.1). The pages are
in `content/docs/`: getting started, configuration, architecture, operations,
security, development.

`.github/workflows/docs.yml` builds on every pull request and deploys to
GitHub Pages on every push to `main`. The slurmstack source repository is
private; these pages are the public description of it and are kept in step with
its code by its pull requests.

## Local preview

```sh
git submodule update --init
hugo server      # http://localhost:1313/docs/
```
