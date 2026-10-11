### Day1

##### docker, kind, kubectl, helm env

config env
```
cat > .devcontainer/devcontainer.json <<'EOF'
{
  "name": "k8s-chaos-lab",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu-22.04",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": { "moby": false },
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": { "minikube": "none" },
    "ghcr.io/devcontainers/features/python:1": {}
  },
  "postCreateCommand": "curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64 && chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind",
  "hostRequirements": { "cpus": 4 }
}
EOF

cat .devcontainer/devcontainer.json
```

两处改动：
- base:ubuntu → base:ubuntu-22.04（固定版本）
- docker-in-docker 加了 "moby": false（改用 Docker 官方源）
否则报错：
```
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ grep -inE "error|failed|denied|not found|unable" /workspaces/.codespaces/.persistedshare/creation.log | head -40
146:#12 0.243 ERROR: Feature "Docker (Docker-in-Docker)" (ghcr.io/devcontainers/features/docker-in-docker) failed to install! Look at the documentation at ********/devcontainers/features/tree/main/src/docker-in-docker for help troubleshooting this error.
147:#12 ERROR: process "/bin/sh -c cp -ar /tmp/build-features-src/docker-in-docker_0 /tmp/dev-container-features  && chmod -R 0755 /tmp/dev-container-features/docker-in-docker_0  && cd /tmp/dev-container-features/docker-in-docker_0  && chmod +x ./devcontainer-features-install.sh  && ./devcontainer-features-install.sh  && rm -rf /tmp/dev-container-features/docker-in-docker_0" did not complete successfully: exit code: 1
159:0.243 ERROR: Feature "Docker (Docker-in-Docker)" (ghcr.io/devcontainers/features/docker-in-docker) failed to install! Look at the documentation at ********/devcontainers/features/tree/main/src/docker-in-docker for help troubleshooting this error.
173:ERROR: failed to build: failed to solve: process "/bin/sh -c cp -ar /tmp/build-features-src/docker-in-docker_0 /tmp/dev-container-features  && chmod -R 0755 /tmp/dev-container-features/docker-in-docker_0  && cd /tmp/dev-container-features/docker-in-docker_0  && chmod +x ./devcontainer-features-install.sh  && ./devcontainer-features-install.sh  && rm -rf /tmp/dev-container-features/docker-in-docker_0" did not complete successfully: exit code: 1
175:2026-10-11 08:02:03.573Z: Error: Command failed: docker buildx build --load --build-arg BUILDKIT_INLINE_CACHE=1 --build-context dev_containers_feature_content_source=/tmp/devcontainercli-root/container-features/0.83.3-1791705690957 --build-arg _DEV_CONTAINERS_BASE_IMAGE=mcr.microsoft.com/devcontainers/base:ubuntu --build-arg _DEV_CONTAINERS_IMAGE_USER=root --build-arg _DEV_CONTAINERS_FEATURE_CONTENT_SOURCE=dev_container_feature_content_temp --target dev_containers_target_stage -f /tmp/devcontainercli-root/container-features/0.83.3-1791705690957/Dockerfile.extended -t vsc-k8s-chaos-lab-f3620039dda6b354cbaa0a8779f9f6838431bee418f57c36425cbb8e85700929-features /var/lib/docker/codespacemount/.persistedshare/empty-folder
176:2026-10-11 08:02:03.573Z: {"outcome":"error","message":"Command failed: docker buildx build --load --build-arg BUILDKIT_INLINE_CACHE=1 --build-context dev_containers_feature_content_source=/tmp/devcontainercli-root/container-features/0.83.3-1791705690957 --build-arg _DEV_CONTAINERS_BASE_IMAGE=mcr.microsoft.com/devcontainers/base:ubuntu --build-arg _DEV_CONTAINERS_IMAGE_USER=root --build-arg _DEV_CONTAINERS_FEATURE_CONTENT_SOURCE=dev_container_feature_content_temp --target dev_containers_target_stage -f /tmp/devcontainercli-root/container-features/0.83.3-1791705690957/Dockerfile.extended -t vsc-k8s-chaos-lab-f3620039dda6b354cbaa0a8779f9f6838431bee418f57c36425cbb8e85700929-features /var/lib/docker/codespacemount/.persistedshare/empty-folder","description":"An error occurred setting up the container."}
187:====================================== ERROR ====================================
188:2026-10-11 08:02:03.588Z: Failed to create container.
190:2026-10-11 08:02:03.588Z: Error: Command failed: docker buildx build --load --build-arg BUILDKIT_INLINE_CACHE=1 --build-context dev_containers_feature_content_source=/tmp/devcontainercli-root/container-features/0.83.3-1791705690957 --build-arg _DEV_CONTAINERS_BASE_IMAGE=mcr.microsoft.com/devcontainers/base:ubuntu --build-arg _DEV_CONTAINERS_IMAGE_USER=root --build-arg _DEV_CONTAINERS_FEATURE_CONTENT_SOURCE=dev_container_feature_content_temp --target dev_containers_target_stage -f /tmp/devcontainercli-root/container-features/0.83.3-1791705690957/Dockerfile.extended -t vsc-k8s-chaos-lab-f3620039dda6b354cbaa0a8779f9f6838431bee418f57c36425cbb8e85700929-features /var/lib/docker/codespacemount/.persistedshare/empty-folder
191:2026-10-11 08:02:03.589Z: Error code: 1302 (UnifiedContainersErrorFatalCreatingContainer)
193:====================================== ERROR ====================================
194:2026-10-11 08:02:03.597Z: Container creation failed.
207:2026-10-11 08:02:11.192Z: Unable to find image 'mcr.microsoft.com/devcontainers/base:alpine' locally
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $
```

上面的错误：失败的是 docker-in-docker 这个官方 feature，不是 kind 那个第三方的。而且它在 0.2 秒内就退出了，说明安装脚本一开头就报错，常见原因是基础镜像 base:ubuntu 指向了很新的 Ubuntu 版本，Docker 的软件源还没有支持它，所以脚本检测发行版时直接失败。

日志里看不到脚本具体打印了什么，所以这个判断是推测，但修法成本很低：把基础镜像固定到一个成熟的版本（Ubuntu 22.04），并让 Docker 从官方 Docker CE 源安装。

##### push & rebuild env
```
git add .
git commit -m "Pin devcontainer base to ubuntu-22.04; use Docker CE in docker-in-docker"
git push
```

##### verify
```
docker ps
kind version
kubectl version --client
helm version
```

```
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ 
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ kind version
kind v0.24.0 go1.22.6 linux/amd64
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ 
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ kubectl version --client
Client Version: v1.37.1
Kustomize Version: v5.8.1
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ helm version
version.BuildInfo{Version:"v4.3.0", GitCommit:"bec5b06ed841fe5269972d864d5177944fd5970f", GitTreeState:"clean", GoVersion:"go1.27.1", KubeClientVersion:"v1.37"}
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $
```

##### Cluster config
```
mkdir -p cluster apps monitoring chaos analyzer docs

cat > cluster/kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

kind create cluster --name chaos-lab --config cluster/kind-config.yaml
kubectl get nodes
```
首次要拉节点镜像，可能 2～5 分钟。成功会看到 3 个节点，状态都是 Ready。

##### Deploy
```
cat > apps/web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
EOF

kubectl apply -f apps/web.yaml
kubectl get pods -o wide
```
`-o wide` 会显示每个 pod 跑在哪个节点上。等 3 个 pod 都是 `Running`

#### Kill pod
```
# 终端 A：持续观察
kubectl get pods -w

# 终端 B：杀掉一个 pod（名字换成你看到的）
kubectl delete pod <web-开头的某个-pod-名字>
```

在终端 A 里会看到旧 pod 进入 `Terminating`，同时新 pod 被创建。记下从删除到新 pod 变成 Running 用了几秒

#### Stop a Node
```
docker ps --filter name=chaos-lab
docker stop chaos-lab-worker
kubectl get nodes
kubectl get pods -o wide -w
```

节点会变成 NotReady，但 K8s 默认要等几分钟才会把它上面的 pod 重新调度: 观察用了7s左右。观察够了就恢复：
```
docker start chaos-lab-worker
kubectl get nodes
```


### 为何用CodeSpace
因为本地Mac版本太老，OrbStack不能用：
```
lei:~ Lei$ 
lei:~ Lei$ brew --version
Homebrew 7.0.9
lei:~ Lei$ 
lei:~ Lei$ 
lei:~ Lei$ sw_vers
ProductName:	macOS
ProductVersion:	12.7.6
BuildVersion:	21H1320
lei:~ Lei$ 
lei:~ Lei$ 
lei:~ Lei$ brew install --cask orbstack
Warning: You are using macOS 12.
We (and Apple) do not provide support for this old version.

Homebrew no longer builds bottles for this configuration.
Consider MacPorts, which provides binary packages for this macOS version:
  https://www.macports.org
Error: orbstack: orbstack: This cask does not run on macOS versions older than Sonoma.
Error: orbstack: orbstack: This cask does not run on macOS versions older than Sonoma.
==> `brew cleanup` has not been run in the last 30 days, running now...
Disable this behaviour by setting `HOMEBREW_NO_INSTALL_CLEANUP=1`.
Hide these hints with `HOMEBREW_NO_ENV_HINTS=1` (see `man brew`).
Removing: /Users/Lei/Library/Caches/Homebrew/gettext--0.20.2.tar.xz... (9.5MB)
Removing: /Users/Lei/Library/Caches/Homebrew/git--2.27.0.tar.xz... (6.1MB)
Removing: /Users/Lei/Library/Caches/Homebrew/git--html--2.27.0.tar.xz... (1.3MB)
Removing: /Users/Lei/Library/Caches/Homebrew/pcre2--10.35.tar.bz2... (1.7MB)
Removing: /Users/Lei/Library/Caches/Homebrew/gdbm--1.18.1.tar.gz... (941.9KB)
Removing: /Users/Lei/Library/Caches/Homebrew/portable-ruby-2.6.3_2.yosemite.bottle.tar.gz... (9.6MB)
Removing: /Users/Lei/Library/Caches/Homebrew/portable-ruby-4.0.7.big_sur.bottle.tar.gz... (12.2MB)
Removing: /Users/Lei/Library/Caches/Homebrew/git--man--2.27.0.tar.xz... (470.7KB)
Removing: /Users/Lei/Library/Caches/Homebrew/git--Net::SMTP::SSL--1.04.tar.gz... (2.5KB)
Removing: /Users/Lei/Library/Caches/Homebrew/pkg-config--0.29.2.tar.gz... (2MB)
Pruned 11 symbolic links from /usr/local

This is a Tier 3 configuration:
  https://docs.brew.sh/Support-Tiers#tier-3
You can report issues with Tier 3 configurations to Homebrew/* repositories!
  
Read the above document before opening any issues or PRs.
lei:~ Lei$ 
```

### Kubernetes 不动
```
@lei-hsia ➜ /workspaces/k8s-chaos-lab (main) $ kubectl get pods -o wide -w
NAME                   READY   STATUS    RESTARTS   AGE    IP           NODE                NOMINATED NODE   READINESS GATES
web-555d94684b-6g7rl   1/1     Running   0          109s   10.244.2.3   chaos-lab-worker2   <none>           <none>
web-555d94684b-hchwp   1/1     Running   0          4m9s   10.244.2.2   chaos-lab-worker2   <none>           <none>
web-555d94684b-n42pf   1/1     Running   0          4m9s   10.244.1.2   chaos-lab-worker    <none>           <none>
web-555d94684b-n42pf   1/1     Running   0          4m33s   10.244.1.2   chaos-lab-worker    <none>           <none>
```
这个不动了

这不是卡住，是 K8s 在等。这正是你要观察的现象，也是面试里值得讲的点。

为什么没动

你停掉了 `chaos-lab-worker`，上面跑着 `web-555d94684b-n42pf`; K8s 的处理有两段等待：

约 40 秒：控制平面收不到节点心跳，才把节点标记为 NotReady。
默认再等 300 秒（5 分钟）：节点被打上"不可达"污点后，pod 默认能容忍这个污点 5 分钟，之后才会被驱逐并在其他节点重新创建。

所以总共大约 5～6 分钟，pod 状态才会变化。-w 是只在状态变化时才输出新行，所以这期间屏幕不动是正常的。日志里 4m33s 那行很可能就是节点失联后 pod 状态更新的那一次。

现在怎么做

再开一个终端，看节点和污点：
```
kubectl get nodes
kubectl describe node chaos-lab-worker | grep -A3 -i taints
```

能看到 `chaos-lab-worker` 是 `NotReady`，并带有 `node.kubernetes.io/unreachable` 污点

#### 实验结束后恢复
```
docker start chaos-lab-worker
kubectl get nodes
kubectl get pods -o wide
```
节点回来后，那个 `Terminating` 的旧 pod 会被清理掉


docker stop 的时间，节点变成 `NotReady` 的时间（约 40 秒），新 pod 创建的时间（约 5 分钟）。
默认 5 分钟的容忍时间是可以调的（`tolerationSeconds`），生产环境里怎么权衡：太短会因网络抖动误驱逐，太长则故障恢复慢。这是个很好的面试话题，也呼应你在 UBS 做 DC 切换演练的经历：主备切换同样存在"多久判定故障、多久切换"的取舍。



