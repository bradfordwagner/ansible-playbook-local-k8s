# Architecture

This repo is a thin Ansible wrapper around the external `bradfordwagner.gitops-toolkit` role. It
supplies configuration and a couple of post-provisioning steps to bootstrap a local k3d cluster with
ArgoCD, then (optionally) hands off application deployment to ArgoCD `ApplicationSet`s.

## Flow

```mermaid
flowchart TD
    subgraph Entry["task run_playbook"]
        PB[playbook.yml]
    end

    CFG[config.yml] -->|vars_files| PB

    PB --> R1["role: bradfordwagner.gitops-toolkit\n(pinned in requirements.yml)"]
    PB --> T1[tasks/auto_deploy.yml]
    PB --> T2[tasks/vault_config.yml]

    subgraph Role["gitops-toolkit role (external, ./roles)"]
        R1 --> K3D["k3d cluster create\n(gtk_clusters.dev, port 8081->80)"]
        R1 --> CERTS["mkcert-issued TLS certs\n(argocd-server-tls, vault-tls)"]
        R1 --> ARGOCD["Install ArgoCD\n(gtk_argocd, via GitOps repo\ndeploy-argocd.git)"]
        CERTS --> CACHE[".cache/rootCA.pem"]
    end

    K3D --> CLUSTER[(k3d cluster: dev)]
    ARGOCD --> CLUSTER

    T1 -->|"kubectl apply -f\nfor each auto_deploy.appsets entry"| APPSET["ArgoCD ApplicationSet\n(e.g. appsets/vault.appset.yaml)"]
    APPSET -->|deploys via ArgoCD| VAULT["Vault (vault-helm chart)\nexposed at /vault, /ui"]

    CACHE -->|copy| T2
    T2 --> VCA["~/.vault-ca\n(local trust for Vault/ArgoCD certs)"]

    CLUSTER -.contains.- ARGOCD
    CLUSTER -.contains.- VAULT
```

## Components

| Component | Source | Role |
|---|---|---|
| `playbook.yml` | this repo | Orchestrates the three stages below against `hosts: localhost`. |
| `config.yml` | this repo | Single source of truth for cluster topology, ArgoCD install source, and which appsets to auto-deploy. |
| `bradfordwagner.gitops-toolkit` role | external, pinned via `requirements.yml`, installed into `./roles` | Creates the k3d cluster(s), issues local TLS certs, installs ArgoCD from a GitOps repo. |
| `tasks/auto_deploy.yml` | this repo | `kubectl apply`s any ArgoCD `ApplicationSet` manifests listed in `config.yml:auto_deploy.appsets`. |
| `appsets/vault.appset.yaml` | this repo | Example `ApplicationSet` — deploys HashiCorp Vault via ArgoCD once opted into `auto_deploy.appsets`. |
| `tasks/vault_config.yml` | this repo | Copies the cluster's root CA (`.cache/rootCA.pem`) to `~/.vault-ca` so local tools trust Vault/ArgoCD TLS. |

## Notes

- Everything under `Role` runs inside the external role's own tasks — its internals aren't in this
  repo; consult `./roles/bradfordwagner.gitops-toolkit` after running `ansible-galaxy install -r
  requirements.yml` if you need to trace behavior further.
- `auto_deploy.appsets` is empty by default in `config.yml`; the Vault appset is opt-in.
