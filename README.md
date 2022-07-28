# Tekton + ArgoCD

## Software
- Kubernetes: v1.23.7
- tekton pipeline: v0.35.1
- argocd: v2.3.3

## Architecture
<!image> 
## Installation
### Tekton
1. Install tekton pipeline
` kubelctl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.35.1/release.yaml`
2. Install tkn cli
```
# Get the file tar.gz
curl -LO https://github.com/tektoncd/cli/releases/download/v0.22.0/tkn_0.22.0_Linux_x86_64.tar.gz
# Extract tkn to your PATH (e.g. /usr/local/bin)
sudo tar xvzf tkn_0.22.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn

```
3. Create Task
