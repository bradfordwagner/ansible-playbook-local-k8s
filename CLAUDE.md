# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An Ansible playbook that bootstraps a local Kubernetes GitOps environment using k3d. It is a thin
consumer of the external `bradfordwagner.gitops-toolkit` Ansible role (pinned in `requirements.yml`),
which does the actual heavy lifting of creating k3d clusters and installing ArgoCD. This repo supplies
the configuration (`config.yml`) and a couple of post-provisioning tasks layered on top.

## Commands

Task runner is [Taskfile.yml](Taskfile.yml) (`task` CLI, go-task):

- `task run_playbook` — runs `ansible-playbook ./playbook.yml`, the main entry point.
- `task delete` — `k3d cluster delete --all`, tears down all k3d clusters.
- `task clean` — removes the `.cache` directory.

Before the first run, install the pinned role dependency:

```
ansible-galaxy install -r requirements.yml
```

This installs `bradfordwagner.gitops-toolkit` into `./roles` (gitignored, not vendored in this repo).

## Architecture

Execution flow, in order, all against `hosts: localhost` ([playbook.yml](playbook.yml)):

1. **`bradfordwagner.gitops-toolkit` role** — external role, version-pinned in `requirements.yml`.
   Reads all `gtk_*` variables from `config.yml` to create k3d cluster(s) and install ArgoCD via GitOps
   (pulling manifests from the `gtk_argocd.manifest.git.repo` repo). Since this role lives outside the
   repo, consult its installed source under `./roles/` (after `ansible-galaxy install`) to understand
   what a given `gtk_*` variable actually does — don't guess from `config.yml` alone.
2. **`tasks/auto_deploy.yml`** — `kubectl apply -f` for each path in `auto_deploy.appsets` (currently
   empty list in `config.yml`; ArgoCD `ApplicationSet` manifests like [appsets/vault.appset.yaml](appsets/vault.appset.yaml)
   are opt-in by uncommenting entries there).
3. **`tasks/vault_config.yml`** — copies `.cache/rootCA.pem` (produced by the gitops-toolkit role's
   `certs` handling) to `~/.vault-ca` for local trust of the self-signed Vault/ArgoCD certs.

`config.yml` is the single source of truth for cluster topology:
- `gtk_clusters.dev` defines one k3d cluster (`dev`), exposing ingress on host port `8081`.
- `gtk_clusters.dev.certs` requests locally-trusted TLS certs (via mkcert, handled by the role) for
  both `argocd-server-tls` and `vault-tls` secrets.
- `gtk_argocd` configures the ArgoCD install itself, including which git repo supplies its manifests.

`appsets/vault.appset.yaml` is an ArgoCD `ApplicationSet` (not applied by Ansible directly, only via
`auto_deploy.appsets`) that deploys HashiCorp Vault from the upstream `vault-helm` chart with TLS
disabled and ingress exposed at `/vault` and `/ui`.

## Notes

- No test suite, linter, or CI config exists in this repo.
- `roles/` and `.cache/` are gitignored — they are generated/installed locally, never commit into them.
