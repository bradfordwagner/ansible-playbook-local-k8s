# Ansible Playbook Local Kubernetes

Bootstraps a local Kubernetes GitOps environment using [k3d](https://k3d.io/), ArgoCD, and (optionally)
HashiCorp Vault. This repo is a thin consumer of the external
[`bradfordwagner.gitops-toolkit`](https://github.com/bradfordwagner/ansible.role.gitops-toolkit)
Ansible role, which does the heavy lifting of creating the k3d cluster and installing ArgoCD; this repo
just supplies configuration (`config.yml`) and a couple of post-provisioning tasks.

See [ARCHITECTURE.md](ARCHITECTURE.md) for a diagram and breakdown of how the pieces fit together.

## Prerequisites

- [Ansible](https://docs.ansible.com/) (`ansible-playbook`, `ansible-galaxy`)
- [k3d](https://k3d.io/)
- [Task](https://taskfile.dev/) (`task` CLI)
- `kubectl`
- [mkcert](https://github.com/FiloSottile/mkcert) (used by the `gitops-toolkit` role to issue locally
  trusted TLS certs)

## Usage

Install the pinned role dependency (only needed once, or after bumping the version in
`requirements.yml`):

```shell
task install_role
```

Run the playbook to create the cluster and install ArgoCD:

```shell
task run_playbook
```

Other available tasks:

```shell
task delete   # k3d cluster delete --all
task clean    # remove the .cache directory
```

## Configuration

`config.yml` is the single source of truth for cluster topology, ArgoCD install source, and which
ArgoCD `ApplicationSet`s get auto-deployed. Notably:

- `gtk_clusters.dev` defines a single k3d cluster (`dev`), with ingress exposed on host port `8081`.
- `gtk_argocd` points at the git repo ArgoCD pulls its own manifests from.
- `auto_deploy.appsets` lists `ApplicationSet` manifests (under [appsets/](appsets)) to `kubectl apply`
  after the cluster is up — e.g. uncomment the Vault entry to deploy
  [appsets/vault.appset.yaml](appsets/vault.appset.yaml).

Since `bradfordwagner.gitops-toolkit` lives outside this repo, consult its installed source under
`./roles/` (after `ansible-galaxy install`) if you need to understand what a given `gtk_*` variable does.
