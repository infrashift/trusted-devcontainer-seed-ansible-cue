# trusted-devcontainer-seed-ansible-cue

A starting point for a **remote development workspace**, as a repository. Clone
it, or let the platform clone it for you, and the first push already produces
a devcontainer image the forge's CI builds and publishes -- under your own
namespace -- and a workspace that runs exactly that image.

This is a **seed**: InfraShift publishes one per language as
`github.com/infrashift/trusted-devcontainer-seed-<language>`, the platform
mirrors each into its forge's `devcontainer-seeds` group, readable by every
developer, and builds it there, so a workspace born from a seed boots on the
seed's own image before you have built anything. From then on the repository
is yours.

## What is in it

    .devcontainer/          the environment: the InfraShift trusted ansible-cue
                            template (ansible-core 2.21.3 on Python 3.13, CUE), plus the workspace runtime
                            contract (see its README)
    Makefile, scripts/      the golden workflow -- the interface between this
                            repository and the forge; for a devcontainer
                            repository it validates rather than compiles
    .gitignore              refuses key material and SSH configuration -- every
                            key you hold is generated in Vault and arrives
                            through the onboarding bundle, never through git

## The loop

1. Your workspace's project directory already holds a clone of your
   repository (the platform cloned it on first boot). Open it in VS Code.
2. Change `.devcontainer/` -- add a feature, bump a version, install a tool in
   the `Containerfile` -- or anything else. Run `make all` to validate before
   pushing: `build` proves `devcontainer.json` parses and names a Containerfile
   that exists, `test` proves every image reference is digest-pinned.
3. Push a branch and open a merge request. The forge's CI builds the
   devcontainer from the merge request's head and, once merged, publishes it as
   `<your namespace>/<repository>-devcontainer:<revision>`.
4. Deploy it to your workspace yourself, from the onboarding bundle:

       ./workspace-ctl.sh images <workspace>
       ./workspace-ctl.sh deploy <workspace> <revision>

   Your project directory survives the redeploy; the image underneath it is
   the one that was reviewed.

## Package registries

Nothing to configure. The platform renders the Nexus package-registry
configuration into your login shell (`/etc/profile.d/package-proxies.sh`):
`pip` and `uv` resolve against the PyPI group with your own credential. There is
no Ansible Galaxy proxy: install collections from git, or as Python packages
where a publisher ships them; `ansible-galaxy` otherwise stays offline. See
`RDW-DEV-GUIDE-ANSIBLE-CUE.md`.

## Starting from a different seed

Every seed is a public repository. To move a workspace to another language's
seed, fetch it into your repository and open a merge request, exactly as for
any other change:

    git remote add seed https://github.com/infrashift/trusted-devcontainer-seed-<language>.git
    git fetch seed
    git merge --allow-unrelated-histories seed/main   # or copy the files you want

## How it is built

Not by `make image`. The forge's CI runs the Dev Container CLI on rootless
podman, which builds `.devcontainer/devcontainer.json`; the result is published
under the repository owner's namespace, tagged with the revision.

## No companion service, on purpose

This seed declares no `.shadowforge/services.json`: an Ansible workspace's
companion is a target host, not a database. The platform deploys its built-in
PostgreSQL beside the workspace regardless, and `psql` is in the image so the
devpod verify can reach it. Add a `services.json` and a `services/<name>/`
directory when a playbook needs something to talk to; the python seed shows
the shape.
