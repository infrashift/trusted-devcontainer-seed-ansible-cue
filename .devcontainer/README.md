# .devcontainer

Two things in one image, and the split is deliberate — see the header comment in
`Containerfile`.

## What came from the trusted template

    template:  ghcr.io/infrashift/trusted-devcontainer-templates/ansible-cue
    version:   v1.0.1

`devcontainer.json`'s ten `features` and the `FROM` line are that template,
digest-pinned and unmodified. (Until 2026-09-10 `ansible-core` was spelled
`ghcr.io:443/...` because envbuilder installed features in string-sorted order
and would have run it before `bootstrap`; the Dev Container CLI that builds
seeds now installs in declaration order, and `scripts/test.sh` asserts that
`bootstrap` is declared first.) A `devcontainer.json` cannot *reference* a
template at build time -- a template is applied, and what it produced is what
is committed here -- so the provenance is recorded above and regenerated with:

    devcontainer templates apply \
      --template-id ghcr.io/infrashift/trusted-devcontainer-templates/ansible-cue

`scripts/test.sh` proves every reference is still digest-pinned.

## What this repository added, and why each one

| Addition | Why |
| --- | --- |
| `containerUser: user` (uid 1001, **gid 0**) | The template creates `dev` (1001:1001). The platform contract is `user` in group 0, assumed by the portal's `WORKSPACE_SSH_USER`, `sshd_config`, the jobspec's volume-init chown and `/home/user/workspace` in the portal README |
| `openssh-server` | Not in the trusted base. Its absence is a workspace that starts, reports its container healthy, and refuses every connection |
| `entrypoint.sh`, `config/sshd_config`, `config/ssh-login.sh` | The workspace runtime contract — the third is the `ForceCommand` the second names |
| `workspace-skel/` | Copied into an EMPTY host volume by the jobspec's prestart task |
| `postgresql` (the `psql` client) | This seed declares no companion service, so the devpod root deploys its built-in database beside the workspace and the verify reaches it with `psql` from inside |
## What the image carries, for the devpod verify

`make verify` in the devpod root asks the image which tools it declares
(check 6). This image's list:

    WORKSPACE_TOOLS=ansible,ansible-galaxy,python3.13,cue,jq,yq,git,git-lfs,syft,grype

## The three copies

`entrypoint.sh`, `config/sshd_config` and `config/ssh-login.sh` are copies of
`terraform/live/devpod-vscode/container/config/`. That is a
real cost of decision 1 — the repository carries the contract so that
envbuilder's output is directly runnable — and it means these three files can
drift from the platform's. **If the devpod root's copies change, these must
change with them.** `make lint` at the Terraform level (`lint-workspace-contract`)
compares every seed's and example's copy against the devpod root's byte for
byte, so the drift is caught in the collection -- but not in a repository
already on the forge, which carries its own copy.
