## [Document](https://minikube.sigs.k8s.io/docs/start/)

### Architecture
```text
----MacOS
    |
    ----docker
        |
        ----registry
        |
        ----gitlab
        |
        ----gitlab-runner:docker
        |
        ----minikube
            |
            ----gitlab-runner:kubernetes
```

## Service

### Installation
```cmd
brew install minikube
```

### Start
> [[Official] Docker Driver](https://minikube.sigs.k8s.io/docs/drivers/docker/)
> [Private Registry Troubleshooting](https://zhuanlan.zhihu.com/p/261722859)

```cmd
# using docker driver with command
minikube start --driver=docker

# using docker driver with config
minikube config set driver docker
minikube start

# using private docker registry
minikube start --insecure-registry=registry.localhost.com:5000

# minikube exist must delete when restart to enable flags
minikube delete
```

### Dashboard
> [Reference](https://github.com/kubernetes/minikube/blob/0c616a6b42b28a1aab8397f5a9061f8ebbd9f3d9/README.md#dashboard)

```cmd
minikube dashboard
```

### Gitlab Stack
```cmd
docker network create dev
docker-compose up -d
```

#### Edit Gitlab Config
```rb
registry_external_url 'http://registry.localhost.com:5000'
```

#### Edit Gitlab Runner Config
```toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "docker-privileged-builder"
  url = "http://gitlab.localhost.com"
  token = "T-5dci8VQ5nJwEZcqGoB"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "docker"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    extra_hosts=["gitlab.localhost.com:172.17.0.3", "registry.localhost.com:172.17.0.5"]
```

## Workround With Docker Private Registry

### Docker Config
> ~/.docker/config.json

```json
{
	"auths": {
		"localhost:5000": {},
		"<private_registry_domain>:5000": {
			"auth": "<username:password|base64>"
		}
	},
	"credsStore": "desktop"
}
```

### [Set Secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials)
```cmd
kubectl create secret generic <secret_name> \
    --from-file=.dockerconfigjson=<path_to_config.json> \
    --type=kubernetes.io/dockerconfigjson
```

### [Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-pod-that-uses-your-secret)
```yml
spec:
  imagePullSecrets:
    - name: <secret_name>
```

## Install Gitlab Runner On K8S Cluster
### [Installation Helm](https://helm.sh/docs/intro/install/#from-homebrew-macos)
```cmd
brew install helm
```

### Create K8S Namespace
> ```kubectl create -f <namespace_yml>```

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: gitlab-runner
  labels:
    name: gitlab-runner
```

### Create Role For Gitlab Runner
> ```kubectl create -f <role_yml>```

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner
  namespace: gitlab-runner
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "secrets", "configmaps", "deployments", "services"]
    verbs: ["list", "get", "watch", "create", "delete", "update", "patch"]
  - apiGroups: ["", "apps"]
    resources: ["pods/exec", "pods/attach"]
    verbs: ["create", "patch", "delete"]
  - apiGroups: ["", "apps"]
    resources: ["pods/log"]
    verbs: ["get"]
```

> ```kubectl describe role --namespace=<namespace_name> <role_name>```

```cmd
Name:         gitlab-runner
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  pods/attach       []                 []              [create patch delete]
  pods/exec         []                 []              [create patch delete]
  pods.apps/attach  []                 []              [create patch delete]
  pods.apps/exec    []                 []              [create patch delete]
  pods/log          []                 []              [get]
  pods.apps/log     []                 []              [get]
  configmaps        []                 []              [list get watch create delete update patch]
  deployments       []                 []              [list get watch create delete update patch]
  pods              []                 []              [list get watch create delete update patch]
  secrets           []                 []              [list get watch create delete update patch]
  services          []                 []              [list get watch create delete update patch]
  configmaps.apps   []                 []              [list get watch create delete update patch]
  deployments.apps  []                 []              [list get watch create delete update patch]
  pods.apps         []                 []              [list get watch create delete update patch]
  secrets.apps      []                 []              [list get watch create delete update patch]
  services.apps     []                 []              [list get watch create delete update patch]
```

### Binding Role
> Gitlab runner is installed in this namespace is going to use the service account ```system:serviceaccount:gitlab-runner:default``` (that is a service account under gitlab-runner namespace)

> ```kubectl create rolebinding --namespace=<namespace_name> <binding_name> --role=<role_name> --serviceaccount=gitlab-runner:default```

> ```kubectl get rolebinding --namespace=<namespace_name>```

```cmd
NAME                    AGE
gitlab-runner-binding   4h16m
```

### Install Gitlab Runner With Helm
```cmd
# add repo
helm repo add gitlab https://charts.gitlab.io

# install runner
helm install --namespace <namespace_name> <name> gitlab/gitlab-runner \
--set gitlabUrl=http://<gitlab_host> \
--set runnerRegistrationToken=<gitlab_runner_token> \
--set rbac.serviceAccountName=default \
--set runners.tags=<tag>

# auto create
NAME                                       READY   STATUS    RESTARTS       AGE
pod/gitlab-runner-7f74ccdb59-779mx         1/1     Running   2 (105m ago)   3h14m

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gitlab-runner         1/1     1            1           3h14m

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/gitlab-runner-7f74ccdb59         1         1         1       3h14m
```

### Uninstall Runner With Helm
> ```helm delete --namespace=<namespace_name> <name>```

## Tips
### [Minikube Useful Command](https://github.com/kubernetes/minikube/blob/0c616a6b42b28a1aab8397f5a9061f8ebbd9f3d9/README.md#reusing-the-docker-daemon)
##### Get Logs
> ```minikube logs```

##### Set Registry Secret
> [[Official] Registry Document](https://minikube.sigs.k8s.io/docs/handbook/registry/)
> ```minikube addons configure registry-creds```
> ```minikube addons enable registry-creds```

```cmd
Do you want to enable AWS Elastic Container Registry? [y/n]: n

Do you want to enable Google Container Registry? [y/n]: n

Do you want to enable Docker Registry? [y/n]: y
-- Enter docker registry server url: registry.localhost.com:5000
-- Enter docker registry username: docker
-- Enter docker registry password:

Do you want to enable Azure Container Registry? [y/n]: n
✅  registry-creds was successfully configured
```

##### [Addons List](https://github.com/kubernetes/minikube/blob/0c616a6b42b28a1aab8397f5a9061f8ebbd9f3d9/README.md#add-ons)
> ```minikube addons list```

```cmd
|-----------------------------|----------|--------------|--------------------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |           MAINTAINER           |
|-----------------------------|----------|--------------|--------------------------------|
| ambassador                  | minikube | disabled     | 3rd party (Ambassador)         |
| auto-pause                  | minikube | disabled     | Google                         |
| csi-hostpath-driver         | minikube | disabled     | Kubernetes                     |
| dashboard                   | minikube | disabled     | Kubernetes                     |
| default-storageclass        | minikube | enabled ✅   | Kubernetes                     |
| efk                         | minikube | disabled     | 3rd party (Elastic)            |
| freshpod                    | minikube | disabled     | Google                         |
| gcp-auth                    | minikube | disabled     | Google                         |
| gvisor                      | minikube | disabled     | Google                         |
| headlamp                    | minikube | disabled     | kinvolk.io                     |
| helm-tiller                 | minikube | disabled     | 3rd party (Helm)               |
| inaccel                     | minikube | disabled     | InAccel <info@inaccel.com>     |
| ingress                     | minikube | disabled     | 3rd party (unknown)            |
| ingress-dns                 | minikube | disabled     | Google                         |
| istio                       | minikube | disabled     | 3rd party (Istio)              |
| istio-provisioner           | minikube | disabled     | 3rd party (Istio)              |
| kong                        | minikube | disabled     | 3rd party (Kong HQ)            |
| kubevirt                    | minikube | disabled     | 3rd party (KubeVirt)           |
| logviewer                   | minikube | disabled     | 3rd party (unknown)            |
| metallb                     | minikube | disabled     | 3rd party (MetalLB)            |
| metrics-server              | minikube | disabled     | Kubernetes                     |
| nvidia-driver-installer     | minikube | disabled     | Google                         |
| nvidia-gpu-device-plugin    | minikube | disabled     | 3rd party (Nvidia)             |
| olm                         | minikube | disabled     | 3rd party (Operator Framework) |
| pod-security-policy         | minikube | disabled     | 3rd party (unknown)            |
| portainer                   | minikube | disabled     | Portainer.io                   |
| registry                    | minikube | disabled     | Google                         |
| registry-aliases            | minikube | disabled     | 3rd party (unknown)            |
| registry-creds              | minikube | enabled ✅   | 3rd party (UPMC Enterprises)   |
| storage-provisioner         | minikube | enabled ✅   | Google                         |
| storage-provisioner-gluster | minikube | disabled     | 3rd party (unknown)            |
| volumesnapshots             | minikube | disabled     | Kubernetes                     |
|-----------------------------|----------|--------------|--------------------------------|
```

##### Config
> ```minikube config view```

```cmd
- driver: docker
```

##### Service
> ```minikube service <service_name>```
> ```minikube service <service_name> --url```

```cmd
|-----------|-------------------------|-------------|---------------------------|
| NAMESPACE |          NAME           | TARGET PORT |            URL            |
|-----------|-------------------------|-------------|---------------------------|
| default   | k8s-demo-service-manual | port-1/80   | http://192.168.49.2:32115 |
|           |                         | port-2/443  | http://192.168.49.2:30000 |
|-----------|-------------------------|-------------|---------------------------|
🏃  Starting tunnel for service k8s-demo-service-manual.
|-----------|-------------------------|-------------|------------------------|
| NAMESPACE |          NAME           | TARGET PORT |          URL           |
|-----------|-------------------------|-------------|------------------------|
| default   | k8s-demo-service-manual |             | http://127.0.0.1:62693 |
|           |                         |             | http://127.0.0.1:62694 |
|-----------|-------------------------|-------------|------------------------|
[default k8s-demo-service-manual  http://127.0.0.1:62693
http://127.0.0.1:62694]
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.

^C✋  Stopping tunnel for service k8s-demo-service-manual.
```

##### Rerange NodePort
> default range 3000-32767
> ```minikube start --extra-config=apiserver.ServiceNodePortRange=<number>-<number>```

### [Kubectl Useful Command](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
##### [Get Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
> ```po | pod | pods ```
> ```kubectl get po -A```
> ```kubectl get pods```
> ```kubectl get pods --show-labels```

```cmd
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
default       k8s-demo-pod                       1/1     Running   0             26m
kube-system   coredns-6d4b75cb6d-mkh5t           1/1     Running   0             28m
kube-system   etcd-minikube                      1/1     Running   0             28m
kube-system   kube-apiserver-minikube            1/1     Running   0             28m
kube-system   kube-controller-manager-minikube   1/1     Running   0             28m
kube-system   kube-proxy-hhphz                   1/1     Running   0             28m
kube-system   kube-scheduler-minikube            1/1     Running   0             28m
kube-system   registry-creds-6b884645cf-787bj    1/1     Running   0             10m
kube-system   storage-provisioner                1/1     Running   1 (27m ago)   28m
```



##### [Get Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
> ```no | node | nodes```
> ```kubectl get nodes```

##### [Get Replication Controller](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
> ```kubectl get rc```

##### [Get Replica Set](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
> ```kubectl get rs```

##### [Get Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
> ```deploy | deployment```
> ```kubectl get deployment```

##### [Get Service](https://kubernetes.io/docs/concepts/services-networking/service/)
> ```svc | service```
> ```kubectl get service```

##### [Get Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
> ```kubectl get namespace```

##### [Debug](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#example-debugging-pending-pods)
> ```kubectl describe nodes <node_name>```
> ```kubectl describe rc <rc_name>```
> ```kubectl describe rs <rs_name>```
> ```kubectl describe deployment <deployment_name>```
> ```kubectl describe pod <pod_name>```
> ```kubectl get pod <pod_name> --output=yaml```

##### [Get Pod Logs](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#examine-pod-logs)
> ```kubectl logs <pod_name> <container_name>```

##### [Get Secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#inspecting-the-secret-regcred)
> ```kubectl get secret <secret_name> --output=yaml```

```cmd
apiVersion: v1
data:
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSJsb2NhbGhvc3Q6NTAwMCI6IHt9LAoJCSJyZWdpc3RyeS5sb2NhbGhvc3QuY29tOjUwMDAiOiB7CgkJCSJhdXRoIjogIlpHOWphMlZ5T2xCQWMzTjNNSEprIgoJCX0KCX0sCgkiY3JlZHNTdG9yZSI6ICJkZXNrdG9wIgp9
kind: Secret
metadata:
  creationTimestamp: "2022-07-13T07:10:33Z"
  name: regcred
  namespace: default
  resourceVersion: "23878"
  uid: 752e027c-b33d-4ab3-9d8d-de5d15fccc88
type: kubernetes.io/dockerconfigjson
```

##### Get Secret Specific Value
> ```kubectl get secret <secret_name> --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode```

```cmd
{
	"auths": {
		"localhost:5000": {
			"auth": "ZG9ja2VyOlBAc3N3MHJk"
		},
		"registry.localhost.com:5000": {
			"auth": "ZG9ja2VyOlBAc3N3MHJk"
		}
	}
}
```

##### Manual Forward Pod's Ports
> ```kubectl port-forward <pod_name> <host_port>:<pod_port>```

##### Manual Expose Pod's Ports By Service
> ```kubectl expose pod <pod_name> --type=NodePort --name=<service_name>```
> ```kubectl expose pod <pod_name> --port=<pod_port> --name=<service_name>```
> ```kubectl expose deploy <deploy_name> --type=NodePort --name=<service_name>```

```cmd
NAME                      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
k8s-demo-service-manual   NodePort    10.98.46.25   <none>        80:32115/TCP,443:30000/TCP   9s
```

##### Exec
> ```kubectl exec <pod_name> -- <command>```

##### Manual Add Pod's Labels
> ```kubectl label pod <pod_name> <label_key>=<label_value>```

##### Run Alpine
> ```kubectl run -i --tty alpine --image=alpine --restart=Never -- sh```

##### Scale Replication Controller
> ```kubectl scale --replicas=<number> -f ./<replication_config>.yaml```

##### Delete Replication Controller
> ```kubectl delete rc <replication_name> --cascade=false```
> cascade option false keep pods alive

##### [Rollout](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)
> ```kubectl set image deploy/<deploy_name> <container_name>=<image_name> --record```
> ```kubectl rollout status deploy <deploy_name>```
> ```kubectl rollout history deploy <deploy_name>```
> ```kubectl rollout undo deploy <deploy_name>```
> ```kubectl rollout undo deploy <deploy_name> --to-revision=<revision>```

```cmd
# set
deployment.apps/k8s-demo-deployment image updated

# status
deployment "k8s-demo-deployment" successfully rolled out

# history
deployment.apps/k8s-demo-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         kubectl set image deploy/k8s-demo-deployment k8s-demo-container=registry.localhost.com:5000/k8s/demo-api-dev:latest --record=true
```

## References
- [Kubernetes 30天學習筆記](https://ithelp.ithome.com.tw/users/20103753/ironman/1590)
- [Gitlab CI/CD Intergration K8S](https://medium.com/@ruben.laguna/installing-a-gitlab-runner-on-kubernetes-ac386c924bc8)
- [Use GitLab CI to run a GitLab runner and run a pipeline on Kubernetes](https://www.alibabacloud.com/help/en/container-service-for-kubernetes/latest/use-gitlab-ci-to-run-a-gitlab-runner-and-run-a-pipeline-on-kubernetes)
- [GitLab Runner Helm Chart](https://docs.gitlab.com/runner/install/kubernetes.html)

## Troubleshooting
- [Helm install - Tags not being applied from values.yaml](https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3967#related-issues)
- [GitLab KAS (Agent) - A certificate is required for KAS](https://gitlab.com/gitlab-org/gitlab/-/issues/352284)
- [What kubernetes permissions does GitLab runner kubernetes executor need](https://stackoverflow.com/a/71506047)