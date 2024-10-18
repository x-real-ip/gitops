# GitOps

[![Update Manifest](https://github.com/x-real-ip/gitops/actions/workflows/update-manifest.yaml/badge.svg)](https://github.com/x-real-ip/gitops/actions/workflows/update-manifest.yaml)
![GitHub repo size](https://img.shields.io/github/repo-size/x-real-ip/gitops?logo=Github)
![GitHub commit activity](https://img.shields.io/github/commit-activity/y/x-real-ip/gitops?logo=github)
![GitHub last commit (branch)](https://img.shields.io/github/last-commit/x-real-ip/gitops/main?logo=github)

- [GitOps](#gitops)
  - [Start ArgoCD WebUI](#start-argocd-webui)
  - [Uptime Kuma](#uptime-kuma)

## Start ArgoCD WebUI

```bash
argocd admin dashboard -n argocd
```

## Uptime Kuma

A sqlite query to find and replace a part in the monitor url.

```
UPDATE monitor SET url = REPLACE(url, 'old', 'new') WHERE url LIKE '%old%';
```
