# Tekton + ArgoCD

## Software
* Kubernetes: v1.23.7
* tekton pipeline: v0.35.1
* argocd: v2.3.3

## Architecture
 ![Markdown](https://github.com/marco210/harbor-/tree/main/images/cicd-architecture.png)
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
* git-clone
* golang-build
* hadolint
* kaniko
4. Create Pipeline
```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: golang-pipeline
spec:
  params:
  - name: git_url
    default: https://github.com/marco210/Hello9000
    type: string
  - name: imageUrl
    default: marco210/pipedemo
    type: string
  - name: imageTag
    default: last
    type: string
  workspaces:
    - name: kubegen-ws
    - name: docker-reg-creds
  tasks:
  - name: git-clone
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: kubegen-ws
    params:
    - name: url
      value: $(params.git_url)
  - name: build-go-app
    taskRef:
      name: golang-build
    workspaces:
    - name: source
      workspace: kubegen-ws
    params:
    - name: package
      value: github.com/marco210/Hello9000
    runAfter:
    - git-clone
  - name: dockerfile-lint
    taskRef:
      name: hadolint
    workspaces:
    - name: source
      workspace: kubegen-ws
    runAfter:
    - git-clone
  - name: build-and-push
    taskRef:
      name: kaniko
    workspaces:
    - name: source
      workspace: kubegen-ws
    - name: dockerconfig
      workspace: docker-reg-creds
    params:
    - name: IMAGE
      value: $(params.imageUrl):$(params.imageTag)
    runAfter:
    - dockerfile-lint
```
5. Create Pipeline Run
```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  creationTimestamp: null
  generateName: golang-pipeline-run-
  namespace: default
spec:
  params:
  - name: git_url
    value: https://github.com/marco210/Hello9000
  - name: imageTag
    value: v4
  - name: imageUrl
    value: marco210/pipedemo
  pipelineRef:
    name: golang-pipeline  
  workspaces:
  - name: kubegen-ws
    persistentVolumeClaim:
      claimName: tekton-pvc
  - name: docker-reg-creds
    secret:
      secretName: docker-creds
status: {}
```
6. Secret reference
* Secret `docker-creds`
```
abc
```

## ArgoCD
* Install ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
* Access UI
  * Create ingress controller
  * Disable https argocd ingress
  Edit argocd-server deployment (add `--insecure`)
  ```
  kubectl -n argocd edit deployments.apps argocd-server
  ```
  ![Markdown](https://github.com/marco210/harbor-/tree/main/images/edit-argocd-server-dpl.png)

  * Change nginx ingress controller(add enable-ssl-passthrough)
  ```
  kubectl -n ingress-nginx edit deployments.apps  ingress-nginx-controller
  ```
  ![Markdown](https://github.com/marco210/harbor-/tree/main/images/edit-ingress-controller.png)
  
  * Get password access UI
  `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`
  
  * Create argocd ingress rule
  ```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: argocd-server-ingress
    namespace: argocd
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  spec:
    rules:
    - host: argocd.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: argocd-server
              port:
                number: 8080
  ```
* Create argocd application
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-argo-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/marco210/test9000.git
    targetRevision: HEAD
    path: argocd
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    automated:
      selfHeal: true
      prune: true
```
