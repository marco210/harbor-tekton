# Harbor
Install harbor using Helm
```
#install
helm install --wait harbor -n harbor harbor/harbor --version v1.9.0  -f harbor-values.yaml
# Upgrade
helm upgrade -n harbor harbor harbor/harbor  --version v1.9.0 -f new-harbor-values.yaml
```
# Tekton + ArgoCD

## Software
* Kubernetes: v1.23.7
* tekton pipeline: v0.35.1
* argocd: v2.3.3

## Architecture
![Architecture](./images/new-cicd-architecture.png)
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
* git-clone: clone repository to workspace
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone-test
spec:
  params:
  - name: repo
    type: string
    description: Git repository to be cloned
    default: https://github.com/marco210/test9000
  workspaces: # define workspace
  - name: source
  steps: # define steps for a task
  - name: clone
    image: alpine/git
    workingDir: $(workspaces.source.path) # using workspace to store data.
    command:
      - /bin/sh
      - -c
      - git clone -v $(params.repo) .
```
* build-push-kaniko: build image from Dockerfile and push to pirvate registry
```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-push-kaniko-test
spec:
  params:
    - name: DOCKERFILE
      description: Path to the Dockerfile to build.
      default: ./Dockerfile
    - name: CONTEXT
      description: The build context used by Kaniko.
      default: ./
    - name: DESTINATION
      description: The url of image to push
      default: harbor.io/library/myapp:v1
  workspaces:
    - name: source
  steps:
    - name: build
      image: gcr.io/kaniko-project/executor:debug
      workingDir: $(workspaces.source.path) # using workspace.
      command:
        - /kaniko/executor
      args: [
            "--dockerfile=$(params.DOCKERFILE)",
            "--context=$(params.CONTEXT)",
            "--insecure=true",
            "--skip-tls-verify=true",
            "--destination=$(params.DESTINATION)",
            ]
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
  volumes:
    - name: docker-config
      secret:
        secretName: regcred
        items:
          - key: .dockerconfigjson
            path: config.json
```
* cleanup: Clean up your persistent data
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
 name: cleanup
spec:
  workspaces:
  - name: source
  steps:
  - name: remove-source
    image: registry.access.redhat.com/ubi8/ubi
    command:
    - /bin/bash
    args:
    - "-c"
    - "rm -rf $(workspaces.source.path)/* && rm -rf $(workspaces.source.path)/.git"
```
4. Create Pipeline
```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-and-list-test
spec:
  params:
  - name: git_repo
    default: https://github.com/marco210/test9000.git
    type: string
  - name: image_url
    type: string
  workspaces:
    - name: codebase
  tasks:
    - name: clone
      taskRef:
        name: git-clone-test
      workspaces:
      - name: source
        workspace: codebase
      params:
      - name: repo
        value: $(params.git_repo)
    - name: build-push
      taskRef:
        name: build-push-kaniko-test
      workspaces:
      - name: source
        workspace: codebase
      params:
      - name: DESTINATION
        value: $(params.image_url)
      runAfter:
      - clone
  finally: # should be clean up folder for second run
  - name: clean
    taskRef:
      name: cleanup
    workspaces:
    - name: source
      workspace: codebase
```
5. Create Pipeline Run
```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: demo-pipeline-run-
spec:
  params:
  - name: git_url
    value: https://github.com/marco210/test9000.git
  - name: image_url
    value: harbor.io/library/myapp:v2
  pipelineRef:
    name: clone-and-list-test
  workspaces:
  - name: codebase
    persistentVolumeClaim:
      claimName: tekton-pvc
```
With private repo in github, to clone we need credentials. Here, using basic authen
`Secret credentials`
```
# secret basic-user-pass
apiVersion: v1
kind: Secret
metadata:
  name: basic-user-pass
  annotations:
    tekton.dev/git-0: https://github.com # Described below
type: kubernetes.io/basic-auth
stringData:
  username: marco210
  password: ghp_github-token
```
`ServiceAccount`
```
cat sa-git.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-bot
secrets:
  - name: basic-user-pass
```
`PipelineRun` with private repository
```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: demo-pipeline-run-
spec:
  serviceAccountName: build-bot
  params:
  - name: git_url
    value: https://github.com/marco210/test9000.git
  - name: image_url
    value: harbor.io/library/myapp:v2
  pipelineRef:
    name: clone-and-list-test
  workspaces:
  - name: codebase
    persistentVolumeClaim:
      claimName: tekton-pvc
```
6. Secret reference
* Secret `regcred`
```
# harbor secret crendential
# kubectl create secret docker-registry regcred --docker-server=https://harbor.io --docker-username=admin --docker-password=Harbor12345 --docker-email=admin@harbor.io --dry-run=client -oyaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL2hhcmJvci5pbyI6eyJ1c2VybmFtZSI6ImFkbWluIiwicGFzc3dvcmQiOiJIYXJib3IxMjM0NSIsImVtYWlsIjoiYWRtaW5AaGFyYm9yLmlvIiwiYXV0aCI6IllXUnRhVzQ2U0dGeVltOXlNVEl6TkRVPSJ9fX0=
kind: Secret
metadata:
  creationTimestamp: null
  name: regcred
type: kubernetes.io/dockerconfigjson
```
7. PV,PVC reference
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: tekton
provisioner: kubernetes.io/no-provisioner
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: tekton
spec:
  storageClassName: harbor
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  local:
    path: /home/hadn/tekton-kaniko-build
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/os
          operator: In
          values:
          - linux
  persistentVolumeReclaimPolicy: Delete
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tekton-pvc
spec:
  storageClassName: harbor
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
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
  ![Markdown](./images/edit-argocd-server-dpl.png)

  * Change nginx ingress controller(add enable-ssl-passthrough)
  ```
  kubectl -n ingress-nginx edit deployments.apps  ingress-nginx-controller
  ```
  ![Markdown](./images/edit-ingress-controller.png)
  
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
      nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
      nginx.ingress.kubernetes.io/ssl-passthrough: "true"
      nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
  spec:
    ingressClassName: nginx
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
    automated:  # automated sync by default retries failed attempts 5 times with following delays between attempts ( 5s, 10s, 20s, 40s, 80s ); retry controlled using `retry` field.
      selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).
      prune: true # Specifies if resources should be pruned during auto-syncing ( false by default ).
```

* If application manifests are located in private repository then repository credentials have to be configured. Argo CD supports both HTTP and SSH Git credentials.
- Using argocd CLI: 
```
argocd repo add https://github.com/argoproj/argocd-example-apps --username <username> --password <password>
```
- Using UI: 
Navigate to `Settings/Repositories`. Add informations of repository: `URL`, `username`, `password`

# References
1. https://gist.github.com/alochym01/d104b84aa3e5316682f09185efc8a442
2. techworldwithnana ArgoCD Tutorial for Beginners | GitOps CD for Kubernetes
