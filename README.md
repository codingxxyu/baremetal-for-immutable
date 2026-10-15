# ACP 4.4.1 Bare Metal 交付包

```text
pack:          baremetal-for-immutable 1.4.0   (见 VERSION.yaml)
ACP:           v4.4.1
Kubernetes:    v1.35.6-acp.1   (官方 OS 矩阵仍是 v4.4.0 那一行)
baseline:      非 DR · External LB · ISO 首装 · 首次交付不做 Global Worker
```

现场按本文一步做。这是 **ACP v4.4.1 全新安装**：先在一台传统 OS 引导节点上起 KIND `minialauda`，再用 ISO 装三台 Global 控制面，installer 交接后，Global 才是管理集群。**不要**把压模 apply 到已经在跑的传统 OS Global 上。

4.4.1 是补丁（CLI 会话刷新、MicroOS alive/iptables）。[OS 矩阵](https://docs.alauda.cn/immutable-infra/1.0/en/overview/os-support-matrix.html) 目前没有 v4.4.1 行，组件版本沿用 v4.4.0 行。Core Package / provider tgz / OS 归档必须是 **同一套 4.4.1 产物**，不要混 4.4.0 包。

1.4.0 把仓库地址寿命写死：引导期 `156.3.1.26:11443`，交接后平台 `:11443`。installer 只改写 Cluster，不改写 BaremetalCluster。API VIP 不是仓库。

每一份 YAML 第一行是 `APPLY-ON`。apply 前先对 kubeconfig：

| 阶段 | kubeconfig | 目录 |
|---|---|---|
| 第一部分 | 引导节点 `~/.kube/config`（minialauda） | `global/control-plane/bootstrap/` `global/control-plane/manifests/` |
| 第二部分 | `./global-kubeconfig`（成品 Global） | `global/management/` |
| 第三部分 | 同上，KIND 已拆掉 | `workload/` |

没有渲染器、没有脚本。`config/site.values.local.yaml` 只是备忘，改它不会改 YAML。必须在对应压模里把 `<占位符>` 换成现场值，再 `kubectl apply`。文件里还留着 `<...>` 就不要 apply。

官方 Global 安装 Step 4 给的是 **差分片段，不能直接 apply**。本仓库已经按 Bare Metal 建集群文档把完整对象写好（Registration / SeedImage / Pool 的 name+hostname+networkDevice / 完整 KCP）。按本仓库文件 apply，不要去拼官方那段 fragment。

官方文档：

- Global 安装：https://docs.alauda.cn/immutable-infra/1.0/en/global/install.html
- Bare Metal Provider：https://docs.alauda.cn/immutable-infra/1.0/install/bare-metal.html
- SeedImage API（`status.downloadURL`）：https://docs.alauda.cn/immutable-infra/1.0/en/apis/providers/bare-metal/seedimage.html
- MachineRegistration API（`status.registrationURL`）：https://docs.alauda.cn/immutable-infra/1.0/en/apis/providers/bare-metal/machineregistration.html
- OS 矩阵：https://docs.alauda.cn/immutable-infra/1.0/en/overview/os-support-matrix.html
- ACP 4.4 发行说明（4.4.1 补丁）：https://docs.alauda.io/container_platform/4.4/overview/release_notes.html

```text
etcd:           v3.6.13-acp.0
containerd:     2.2.6-3
coredns:        1.14.3-v4.4.6
pause:          3.10
kube-ovn chart: v4.4.6
architecture:   linux/amd64  (x86-64-v2)
namespace:      cpaas-system
```

占位对照：`config/site.values.example.yaml`。复制为 `config/site.values.local.yaml`（已 gitignore），**只作备忘，不会自动渲染**。

现场改 YAML 的正确做法：

```bash
cp config/site.values.example.yaml config/site.values.local.yaml
# 1. 先把 local 文件里的值填完，当作对照表
# 2. 再在对应 YAML/JSON 里搜索同一个 <占位符> 替换
grep -RInE '<[^>]+>' --include='*.yaml' --include='*.json' .
# 3. apply 前再扫一遍目标文件
grep -nE '<[^>]+>' 你准备 apply 的那个文件
```

---

## 目录

```text
VERSION.yaml CHANGELOG.md
config/site.values.example.yaml
global/control-plane/bootstrap/     # 01–05 只在 minialauda；见该目录 README
global/control-plane/manifests/     # Cluster 名必须是 global；只在 minialauda
global/management/                  # 成品 Global 上装两个 provider；见该目录 README
global/worker/                      # 首次交付跳过。manifests/ = 引导期；post-global/ = 交接后
workload/control-plane/manifests/   # 集群名不能是 global
workload/worker/manifests/
```

| 阶段 | 目录 | kubeconfig |
|---|---|---|
| 引导集群装 provider | `global/control-plane/bootstrap/01`–`03` | minialauda |
| Global 控制面注册 / CAPI | `global/control-plane/manifests/` | minialauda |
| 导入 + installer | `global/control-plane/bootstrap/04`–`05` | minialauda |
| Global 作为管理集群 | `global/management/` | **成品 Global** |
| Workload | `workload/` | **成品 Global** |
| Global Worker | `global/worker/manifests/` 首次 **不要 apply**；交接后用 `global/worker/post-global/` | **成品 Global** |

---

## 硬性约束

1. Cluster / BaremetalCluster **必须** 名叫 `global`。其它资源用 `global-` 前缀。Workload 集群名 **不能** 是 `global`。
2. AppRelease 里 `global.cluster.type` 必须是 `Baremetal`（chart 分类，所有 IaaS 都这样写）。
3. `install.device` 只写 `/dev/elemental-install-target`。真实盘符不要写进 YAML。每台 Live ISO 上 `ln -s`，装完改 BMC 为 disk-first。
4. 控制面 ISO 和 Worker ISO 分开。不要用控制面 ISO 装 Worker。
5. Bare Metal `maxSurge` 必须是 `0`。
6. Registration URL 是凭证。不要贴进工单、聊天、README 副本。
7. 不要 `kubectl delete cluster global` 当清理。交接后只拆除 minialauda KIND。
8. `dcs-import-extra-resources` 不要导入 Elemental plan Secret、kubeadm bootstrap Secret、KubeadmConfig、SeedImage 对象。
9. 本包是 ISO 首装路径。HCS qcow2 + `elemental-register --install --no-toolkit` 不是官方 BM Global/Workload 首装。
10. 不要克隆已注册磁盘当云模板。
11. `global/management/` 禁止 apply 到 minialauda。`bootstrap/01`–`05` 禁止 apply 到成品 Global。
12. 首次交付不要 apply `global/worker/manifests/`（YAML 第一行是 `DO-NOT-APPLY`）。交接后若要加 Global Worker，用 `global/worker/post-global/`。
13. SeedImage `spec.size: 20Gi` 需要默认 StorageClass。PVC Pending 时 ISO 不会出 `downloadURL`。
14. `cpaas.io/registry-address` 只写 `<host>:11443`，不要写 API VIP、不要加 `/v2/`。引导阶段是 `156.3.1.26:11443`；交接后是平台仓库。不要把 `03`/`05` 在 installer 之后再 apply 一遍。

---

## 闸门（过了才能进下一步）

不要凭「差不多了」往下走。每行左边是命令，右边是必须看到的结果。

| 过了才能做 | 检查 | 必须 |
|---|---|---|
| apply Registration / SeedImage | catalog 有 `v1.35.6-acp.1`；BM provider Ready；**默认 StorageClass 能绑定 20Gi PVC** | 第 5 步、进场检查 |
| 挂 ISO | `status.downloadURL` 非空（不要死等某个 condition 名字） | 第 6 步 |
| apply Pool / KCP | 三台 Inventory `Ready=True`，已离开 Live | 第 8–9 步 |
| apply `04` 导入清单 | KCP Ready、三台 Master Node Ready | 第 12 步 |
| POST installer | `04` 已 apply，inventory 名与 Pool 一致；JSON 已校验；curl 在引导节点打 Pod IP:8080 | 第 13–14 步 |
| 推镜像到平台仓库 | installer Success **且** 第 15 步 handoff 三项都过 | 第 15 步 |
| 拆 KIND / apply Workload 之前 | Cluster **和** BaremetalCluster 的 `cpaas.io/registry-address` 已是平台仓库，不是 `156.3.1.26` | 第 15.1 步 |
| apply `global/management/` | 平台仓库已有两个 OS 镜像 + 两个 chart | 第 16 步 |
| 拆 KIND | 第 15 + 15.1 + 16 + 17 都过；`global-kubeconfig` 能 `get nodes` | 第 17 步末 |
| apply Workload | 成品 catalog 有 `v1.35.6-acp.1`，KIND 已拆或不再使用；Global 上也有默认 StorageClass | 第 17 步 catalog |

---

## 需要准备的包

从 Customer Portal 取 **同一套 ACP 4.4.1** 产物。压缩包不进 git，放到引导节点工作目录即可。

| 包 | 用途 |
|---|---|
| `installer-core-v4.4.1-hybrid.tar` | Core Package。`setup.sh` 起 KIND minialauda + 嵌入式仓库 + cpaas-installer |
| `cluster-api-provider-kubeadm.amd64.<kubeadm-provider-chart-version>.tgz` | packtool 推进引导仓库 |
| `cluster-api-provider-baremetal.amd64.<baremetal-provider-chart-version>.tgz` | 同上 |
| `<bare-metal-os-archive>.tar` | Bare Metal OS **OCI 归档**（不是 qcow2）。里面是 `baremetal-base-image-iso` 和 `baremetal-base-image`，**同一个 tag** |

Chart 版本从 tgz 文件名或 `helm show chart` 读，填进：

- `<kubeadm-provider-chart-version>`
- `<baremetal-provider-chart-version>`

OS tag 在导入后读：

```bash
ctr -n bare-metal-os images ls -q
```

把完整 tag 填进所有 `<os-image-tag>`。catalog 缺 `v1.35.6-acp.1` 时 CAPI 会停在 `ImageCatalogMiss`，这是终态错误，不是暂时的。

引导节点：传统 OS（不是 Alauda OS）、静态 IP 整场安装不能变。官方下限 x86_64 **8 核 / 16 GB / 磁盘 ≥ 300 GB**（解压目录 + `/var/cpaas` + `/var/lib/containerd`）。引导节点本身不要求 x86-64-v2；**Alauda OS 物理机**要求 x86-64-v2。

仓库地址不要混。注解格式是 `<host>:11443`，**不要**加 `/v2/`，**不要**写成 API VIP。

| 地址 | 谁用 | 寿命 |
|---|---|---|
| `127.0.0.1:11443` | 引导节点本机 `packtool` / `ctr push` | 引导期 |
| `172.18.0.1:11443` | KIND 里 AppRelease `repoURL`、Global 控制面 SeedImage `baseImage` | 引导期。KIND 拆掉即死 |
| `156.3.1.26:11443` | **首次 apply** Global Cluster / BaremetalCluster 的 `cpaas.io/registry-address`；物理机装 OS 时拉镜像 | 临时。installer 之后必须换成平台仓库 |
| `<platform-access-domain-or-vip>:11443` | installer JSON `registry.domain`；成品 Global / Workload 的注解、AppRelease `repoURL`、SeedImage `baseImage` | 永久。KIND 拆掉后只剩这个 |

官方 installer **只改写 Cluster（以及 DCSCluster）** 的 `cpaas.io/registry-address`。**不改写 BaremetalCluster**。所以：

- Global **Cluster** `05`：第一次写引导 IP。installer Success 后看一眼，应已变成平台仓库。
- Global **BaremetalCluster** `03`：第一次也写引导 IP。Success 后若还是 `156.3.1.26:11443`，**patch** 成平台仓库，不要重新 apply 这份 YAML。
- Workload `03` / `05`：创建时就写平台仓库，永远不要写 `156.3.1.26`。

API VIP（`<global-control-plane-vip>`，TCP 6443）只出现在 `BaremetalCluster.spec.controlPlaneLoadBalancer.host`、KCP etcd SAN、installer `console.host`。它不是仓库地址。

installer JSON 的 `registry.domain` **必须是平台入口:11443**，不要写引导仓库。不要设 `registry.externalAddress`。

---

## 占位符

全文搜索 `<` 一次性改。常用：

| 占位符 | 含义 |
|---|---|
| （已填）`156.3.1.26` | 引导节点 IP。dex.tls SAN、elemental `:12443`、**第一次** Global Cluster/BaremetalCluster 仓库注解、installer `hostIP`。不要再搜 `<bootstrap-host-ip>` |
| `<platform-access-domain-or-vip>` | 平台入口。交接后仓库、elemental `:443`、`console.globalHost`。**可以**和 Global VIP 相同，但仓库端口永远是 `:11443` |
| `<global-control-plane-vip>` | Global API VIP（TCP 6443）。写进 BaremetalCluster `host` / installer `console.host` / etcd SAN。**不是** `cpaas.io/registry-address` |
| `<unique-vrid>` | 仅 Internal Alive。External 时 **删掉** `vrid` 字段 |
| `<os-image-tag>` | OCI 归档里两个镜像共用的 tag |
| `<kubeadm-provider-chart-version>` | kubeadm provider chart 版本 |
| `<baremetal-provider-chart-version>` | BM provider chart 版本 |
| `<ssh-authorized-keys>` | `boot` 用户公钥，完整一行 |
| `<global-machine-inventory-N>` | 注册后的 MachineInventory 名，不要猜 |
| `<base64-platform-admin-password>` | installer JSON 里的平台管理员密码，**必须是 base64**，不是明文 |
| `<registry-username>` / `<base64-registry-password>` | 平台仓库账号；密码同样是 base64 |
| `<workload-cluster-name>` | 不能是 `global` |
| `<workload-control-plane-vip>` | Workload API VIP |
| `<workload-cp-inventory-N>` / `<workload-worker-inventory-1>` | Workload 注册后的 inventory 名 |

CIDR（本包已按现场写入，四处必须一致：Cluster、installer JSON、`site.values`）：

| | Pod | Service | kube-ovn join |
|---|---|---|---|
| Global | `10.3.0.0/16` | `10.4.0.0/16` | `100.64.0.0/16` |
| Workload | `10.5.0.0/16` | `10.6.0.0/16` | `100.65.0.0/16` |

不要把 Global Service 写成 `10.4.0.0/14`：那一段包含 `10.4–10.7`，会和 Workload Pod/Service 重叠。Join 是 `100.64/100.65`，不是 `10.64/10.65`。管理网 `156.3.1.0/24` 不要写进这些字段。

---

# 第一部分：引导节点上创建 Global

所有命令默认在 **引导节点** 上，kubeconfig 指向 minialauda，直到交接完成。

## 0. 现场值与进场检查

```bash
cp config/site.values.example.yaml config/site.values.local.yaml
# 编辑 local 文件，然后用同一组值替换 YAML 里的 <...>
grep -RInE '<[^>]+>' --include='*.yaml' --include='*.json' .
```

进场（引导节点，包已放到 `/root`）：

```bash
hostname -I
free -h
df -h / /var /var/lib/containerd /root
# 磁盘空闲建议：解压目录 ≥100G，/var/cpaas + containerd 还能再放下 OS 归档
ls -lh /root/installer-core-v4.4.1-hybrid.tar \
       /root/cluster-api-provider-kubeadm.amd64.*.tgz \
       /root/cluster-api-provider-baremetal.amd64.*.tgz \
       /root/*.tar
# chart 版本从文件名抄进 YAML
ls /root/cluster-api-provider-*.tgz
# 三台 Global 控制面 BMC 能挂 ISO；VIP:6443 转发规则已建（后端可以还没有）
# 物理机 CPU 是 x86-64-v2（引导节点本身不要求）
# SeedImage spec.size 是 20Gi：KIND 起来之后（第 1 步末）必须有默认 StorageClass
```

SSH 公钥（KCP `boot` 用户，不能空）：

```bash
# 已有钥匙就用现成的 .pub；没有再生成。私钥不要放进仓库。
test -f ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
# 把这一整行填进 06-control-plane.yaml 的 sshAuthorizedKeys
```

installer JSON 密码必须是 **base64 编码后的字符串**（官方字段名就叫 `<base64-...>`）。在引导节点上：

```bash
# 交互输入，避免密码进 shell history
printf 'platform admin password: ' && read -rs PLAIN && echo
printf '%s' "$PLAIN" | base64 | tr -d '\n'; echo
# 把输出填进 05-installer-config-baremetal.json 的 basic.password

printf 'registry password: ' && read -rs REGPLAIN && echo
printf '%s' "$REGPLAIN" | base64 | tr -d '\n'; echo
# 把输出填进 registry.password
```

不要把明文或填好密码的 JSON 提交 git。副本存成引导节点上的 `installer-config.json`（已 gitignore）。

External LB：在 BMC/LB 上确认 `<global-control-plane-vip>:6443` TCP passthrough 到三台 Global 控制面 `:6443`（机器装完才有后端，先把 VIP 和转发规则建好）。

Internal Alive：`03-baremetal-cluster.yaml` 改 `type: Internal`，打开 `vrid`，保证二层能跑 keepalived。

## 1. Core Package → minialauda

引导节点 root：

```bash
mkdir -p /root/cpaas-install
tar -xvf installer-core-v4.4.1-hybrid.tar -C /root/cpaas-install
cd /root/cpaas-install/installer
bash setup.sh
mkdir -p "${HOME}/.local/bin" ~/.kube
if ! command -v kubectl >/dev/null 2>&1; then
  nerdctl cp minialauda-control-plane:/usr/bin/kubectl "${HOME}/.local/bin/kubectl"
  chmod 0755 "${HOME}/.local/bin/kubectl"
  export PATH="${HOME}/.local/bin:${PATH}"
fi
cp /var/cpaas/data/alauda.kubeconfig ~/.kube/config
kubectl get nodes
kubectl -n cpaas-system get pods
```

`setup.sh` 会问引导节点 IP、仓库是否设密码等。按提示填，**HOST_IP 必须是整场不变的静态 IP**，和后面 YAML 里的 `156.3.1.26` 一致。脚本跑完才有 kubeconfig。

成功：KIND 名 `minialauda`；嵌入式仓库本机 `127.0.0.1:11443`；cpaas-installer 在跑。默认仓库匿名，不要造一个空的 `global-registry-auth`。

SeedImage 要 20Gi PVC。KIND 起来后立刻确认有默认 StorageClass，没有就先修再 apply Registration：

```bash
kubectl get sc
kubectl get sc -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{"\n"}{end}'
# 必须打出至少一个名字。空 = ISO 永远没有 downloadURL。
```

`nerdctl cp minialauda-control-plane:...` 依赖 Core Package 带的 nerdctl/containerd。如果 `nerdctl` 不在 PATH：

```bash
ls /root/cpaas-install/installer/res/amd64/ | head
command -v nerdctl || command -v docker
# KIND 控制面容器名一般是 minialauda-control-plane
nerdctl ps --format '{{.Names}}' | grep minialauda || docker ps --format '{{.Names}}' | grep minialauda
```

没有现成 `kubectl` 时，用上面那个容器把二进制拷出来（官方写法是 nerdctl）。已经有匹配版本的 kubectl 就跳过这段。

## 2. 推 provider chart 和 OS 镜像

packtool 打 **本机** 仓库，不要打 `172.18.0.1`。**先看 help**，不要假设认证开关叫 `-u`：

```bash
PACKTOOL=/root/cpaas-install/installer/res/amd64/packtool
test -x "$PACKTOOL"
$PACKTOOL pack push -h
$PACKTOOL pack push -r "127.0.0.1:11443" -c /root/cluster-api-provider-kubeadm.amd64.<kubeadm-provider-chart-version>.tgz
$PACKTOOL pack push -r "127.0.0.1:11443" -c /root/cluster-api-provider-baremetal.amd64.<baremetal-provider-chart-version>.tgz
```

引导仓库默认匿名。`setup.sh` 给仓库设了账号时，按 **刚才 help 里的认证参数** 再推（有的版本是 `-u 用户:密码`，有的是 `--username` / `--password`），并且 AppRelease 要挂 `global-registry-auth`（官方默认匿名路径不要造空 Secret）。

OS 归档不含在 provider 包里，单独导入。`/var/lib/containerd` 至少要有归档那么大的空闲。`bare-metal-os` 是 dedicated namespace；import 报 namespace 不存在时先建再 import。不要先 `ctr images rm`。

```bash
command -v ctr
ctr namespaces ls || true
ctr namespaces create bare-metal-os 2>/dev/null || true
ctr -n bare-metal-os images import --all-platforms --no-unpack /root/<bare-metal-os-archive>.tar
ctr -n bare-metal-os images ls -q
# 应看到同一 tag 的两条：
#   baremetal-base-image-iso:<os-image-tag>
#   baremetal-base-image:<os-image-tag>
export OS_IMAGE_TAG=<os-image-tag>
for repo in baremetal-base-image baremetal-base-image-iso; do
  ctr -n bare-metal-os images push --plain-http \
    "127.0.0.1:11443/tkestack/${repo}:${OS_IMAGE_TAG}" "${repo}:${OS_IMAGE_TAG}"
  curl -s "http://127.0.0.1:11443/v2/tkestack/${repo}/tags/list"
done
```

把这个 tag 填进所有 YAML 的 `<os-image-tag>`。**不要**在这一步 `ctr images rm`：交接后还要从同一个 `bare-metal-os` namespace 再推到成品仓库。

ISO 给 SeedImage；catalog 给每一次 CAPI 重装。只导入 ISO、不填 catalog，控制面会卡在 `ImageCatalogMiss`。

## 3. dex.tls（引导集群，必须带引导节点 IP SAN）

文件：`global/control-plane/bootstrap/01-dex-tls-bootstrap.yaml`

改：

```yaml
spec:
  commonName: 156.3.1.26
  ipAddresses:
    - "156.3.1.26"
```

```bash
kubectl apply -f global/control-plane/bootstrap/01-dex-tls-bootstrap.yaml
kubectl -n cpaas-system get certificate dex-tls-bootstrap
kubectl -n cpaas-system get secret dex.tls
```

`Ready=True` 后再装 BM provider。这个 Secret **不要** 导入成品 Global。

## 4. Kubeadm provider（引导集群）

文件：`global/control-plane/bootstrap/02-kubeadm-provider-apprelease.yaml`

改：`<kubeadm-provider-chart-version>`、`<platform-access-domain-or-vip>`。`repoURL` 保持 `172.18.0.1:11443`。

```bash
kubectl apply -f global/control-plane/bootstrap/02-kubeadm-provider-apprelease.yaml
kubectl -n cpaas-system get apprelease cluster-api-provider-kubeadm
```

## 5. Bare Metal provider（引导集群）

文件：`global/control-plane/bootstrap/03-baremetal-provider-apprelease.yaml`

改：

```yaml
platformUrl: https://156.3.1.26
provider.imageCatalog.images.v1.35.6-acp.1.tag: <os-image-tag>
elemental.server.url: https://156.3.1.26:12443
handoffHook.controlPlaneVIP: <global-control-plane-vip>
```

```bash
kubectl apply -f global/control-plane/bootstrap/03-baremetal-provider-apprelease.yaml
kubectl -n cpaas-system get apprelease cluster-api-provider-baremetal
kubectl -n cpaas-system get deploy cluster-api-provider-baremetal-manager elemental-operator
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

catalog 必须有：

```yaml
data:
  v1.35.6-acp.1: 172.18.0.1:11443/tkestack/baremetal-base-image:<os-image-tag>
```

没有就按官方方式补，不要覆盖其它 key。非 DR 基线：不要在引导 Registration 上设 `baremetal.cluster.io/system-agent-server-url`。

## 6. Global 控制面 Registration / SeedImage

文件：`global/control-plane/manifests/01-registration-seedimage.yaml`

改：`<os-image-tag>`。`baseImage` 必须是引导仓库：

```text
172.18.0.1:11443/tkestack/baremetal-base-image-iso:<os-image-tag>
```

`device` 保持 `/dev/elemental-install-target`。`emulate-tpm: true`（没有硬件 TPM 时）。

```bash
kubectl apply -f global/control-plane/manifests/01-registration-seedimage.yaml
kubectl -n cpaas-system get seedimage global-registration-iso
kubectl -n cpaas-system get pvc -l elemental.cattle.io/seedimage=global-registration-iso 2>/dev/null || kubectl -n cpaas-system get pvc
# PVC Pending = 没有默认 StorageClass，ISO 不会出来。先修 SC，不要干等 30 分钟。
# 闸门是 downloadURL，不是某个固定的 condition 名（Ready 与 SeedImageReady 现场都见过）：
ISO_URL=""
for i in $(seq 1 60); do
  ISO_URL=$(kubectl -n cpaas-system get seedimage global-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
kubectl -n cpaas-system get seedimage global-registration-iso \
  -o jsonpath='{.status.conditions}{"\n"}{.status.state}{"\n"}'
```

`status.downloadURL` 非空才能挂 ISO。官方字段：

| 字段 | 用途 |
|---|---|
| `status.downloadURL` | ISO 下载地址 |
| `status.downloadToken` | 标识这次下载 |
| `status.checksumURL` | 校验和（有就核） |
| `status.registrationURL`（在 **MachineRegistration** 上，不是 SeedImage） | 注册入口，是凭证 |

在 **引导节点**上下 ISO（URL 通常只在集群网络内可访问，不要指望笔记本直连）：

```bash
ISO_URL=$(kubectl -n cpaas-system get seedimage global-registration-iso \
  -o jsonpath='{.status.downloadURL}')
test -n "$ISO_URL"
echo "$ISO_URL"   # 只在引导节点上看，不要贴进工单
curl -fL --retry 3 -o /root/global-registration.iso "$ISO_URL"
ls -lh /root/global-registration.iso
# 可选校验
CHECKSUM_URL=$(kubectl -n cpaas-system get seedimage global-registration-iso \
  -o jsonpath='{.status.checksumURL}')
if [ -n "$CHECKSUM_URL" ]; then
  curl -fsSL "$CHECKSUM_URL"
fi
```

把 ISO 拷到能挂 BMC 虚拟光驱的操作机（U 盘、scp、iDRAC/iLO 共享都可以）。不要用这张 ISO 装 Worker。`cleanupAfterMinutes: 120` 过后构建产物会被清，超时就重新 apply / retrigger，不要假设文件一直在。

Registration URL 给现场看一眼、确认 Live 能注册即可。**不要**写进仓库、工单、聊天、本 README 副本：

```bash
# 只在引导节点终端看，不要重定向进 git 目录
kubectl -n cpaas-system get machineregistration global-registration \
  -o jsonpath='{.status.registrationURL}{"\n"}'
# 需要留在本机时写到 gitignore 的文件：
# kubectl ... > /root/registration-url.txt && chmod 600 /root/registration-url.txt
```

无 DHCP：进 Live 后按第 7.1 步配静态 IP（能访问 `156.3.1.26:12443` 和 `:11443`），再做软链。时钟在进 Live 前从 BMC/firmware 校好，三台相差不超过 10 秒。

## 7. [每台控制面 Live ISO] 配网 → 系统盘软链 → 等安装自己跑

BMC 挂上第 6 步的 ISO，从光驱启动。Live 控制台一般是 root、无密码（以现场 ISO 提示为准）。

### 7.1 无 DHCP 时先配静态 IP

官方要求：没有 DHCP 就先在 Live 控制台配 NIC，再等 MachineInventory。地址、网关、DNS、网卡名按 **这台机器** 改，不要抄别的现场。

```bash
ip -br link
ip -br addr
# 下面 eth0 / 地址只是形状，换成这台控制面的管理网
nmcli con add type ethernet ifname eth0 con-name mgmt \
  ipv4.method manual \
  ipv4.addresses <this-host-ip>/<prefix> \
  ipv4.gateway <gateway> \
  ipv4.dns <dns> \
  ipv6.method ignore \
  connection.autoconnect yes
nmcli con up mgmt
ping -c 3 156.3.1.26
# elemental 和拉镜像都要通
nc -vz 156.3.1.26 12443 || curl -skI "https://156.3.1.26:12443" | head
nc -vz 156.3.1.26 11443 || curl -sI "http://156.3.1.26:11443/v2/" | head
```

有 DHCP 且能 ping 通引导节点就跳过 7.1。VLAN 现场见第三部分 Worker 说明：IP 配在 VLAN 口上，不要配在物理口。

### 7.2 系统盘软链（注意是 `ln`，不是 `ls`）

YAML 的 `install.device` 是 `/dev/elemental-install-target`。真实盘不要写进 YAML。

```bash
lsblk -d -e7 -o NAME,PATH,SIZE,MODEL,SERIAL,WWN,HCTL
ln -sfn /dev/disk/by-id/wwn-<this-host-os-disk> /dev/elemental-install-target
# 盘序稳定时也可以：ln -sfn /dev/sda /dev/elemental-install-target
readlink -f /dev/elemental-install-target
test -b "$(readlink -f /dev/elemental-install-target)" && echo TARGET_IS_BLOCK
```

不要链到数据盘、`sr*`、`loop*`。链接只活在当前 Live 会话。硬盘已有旧系统时先按 FAQ「已有操作系统的磁盘」wipe，再 `ln`。

### 7.3 安装会自己开始

软链完成后 **不要**再手搓一套 `elemental-register`（除非官方 Live 提示要你手动跑）。ISO 已经绑了这张 MachineRegistration；agent 看到块设备后会分区、安装、reboot。

在 Live 上看进度：

```bash
ls -l /dev/elemental-install-target
journalctl -f | grep -Ei 'elemental|register|install' || true
# 有 elemental 日志文件时：
ls /var/log/elemental* /tmp/elemental* 2>/dev/null || true
```

引导节点上对应这台会晚一点出现 Inventory：

```bash
kubectl -n cpaas-system get machineinventories.elemental.cattle.io -w
```

一直不装：确认 7.1 网络、7.2 `TARGET_IS_BLOCK`、Registration URL 能解析到 `https://156.3.1.26:12443`、时钟误差 < 10s。不要在没软链时干等。

安装触发重启后，在 BMC：

1. 弹出/卸载 SeedImage 虚拟介质。
2. 启动顺序改成 **disk-first**。
3. 确认下次进已安装 OS，不是再次进 Live ISO。

已离开 Live ISO 的机器不要再回 Live 重建链接。

## 8. 确认已从系统盘启动

在已安装 OS 上：

```bash
hostname
findmnt -n -o SOURCE,FSTYPE,TARGET /
findmnt /run/initramfs/live || echo NO_LIVE_ISO_ROOT
lsblk -e7 -o NAME,PATH,TYPE,SIZE,MODEL,SERIAL,WWN,FSTYPE,LABEL,MOUNTPOINTS
```

`/` 不能是 `LiveOS_rootfs`；要能看到 `COS_STATE`。仍是 Live 就不要进 Pool。

## 9. 三台 Inventory

在引导节点：

```bash
kubectl -n cpaas-system get machineinventories.elemental.cattle.io -o wide
```

每台：`Ready=True`，allocation 空或 `Available`，owner 空，已从系统盘启动。把真实名字填进 `02-control-plane-pool.yaml`。

`04-dcs-import-extra-resources.yaml` **先不要填、先不要 apply**。等第 12 步 KCP Ready 之后，从 BaremetalMachine 再抄一遍 inventory 名（和 Pool 必须一致），那时再 apply 04。

## 10. Global Worker — 首次交付跳过

读 `global/worker/SKIP.md`。本步 **不要 apply**。三台控制面即可。以后要加 Global Worker，在成品 Global 上按第三部分 Worker 的流程另起 Registration，不要用 bootstrap ISO。

## 11. Control Plane Pool → Template → BaremetalCluster → Cluster → KCP

顺序不要颠倒。Pool 必须在 KCP 之前。

**Pool** `global/control-plane/manifests/02-control-plane-pool.yaml`

官方 Global Step 4 fragment 写成 `- global-cp-1` 这种纯名字列表，**不能 apply**。本文件是完整 shape：

```yaml
spec:
  clusterName: global
  machineInventories:
    - name: <global-machine-inventory-1>   # 第 9 步查到的真实 Inventory 名
      hostname: global-cp-1                # kubeadm 节点名，按现场规划
      networkDevice: eth0                  # 这台的 CNI 网卡
```

某台 CNI 网卡不是 `eth0` 时只改那一台。

**Template** `04-control-plane-machine-template.yaml` — 已指向 `global-control-plane-pool`。

**BaremetalCluster** `03-baremetal-cluster.yaml`

```yaml
metadata:
  name: global
spec:
  controlPlaneLoadBalancer:
    type: External
    host: <global-control-plane-vip>
    port: 6443
    # External：删掉 vrid
  networkDevice: eth0
```

`cpaas.io/registry-address` 在**第一次 apply** 时是 `156.3.1.26:11443`（物理机能打到引导仓库）。installer 只改写 **Cluster**，不改写这份 BaremetalCluster。交接后若还是引导 IP，见第 15.1 步 patch。**installer 之后不要再 apply 03/05。**

**Cluster** `05-cluster.yaml` — `metadata.name: global`，`is-global: "true"`，k8s `v1.35.6-acp.1`，kube-ovn `v4.4.6`。

**KCP** `06-control-plane.yaml`

改：

```yaml
sshAuthorizedKeys:
  - "<ssh-authorized-keys>"
etcd.local.serverCertSANs:
  - "<global-control-plane-vip>"
  - "<platform-access-domain-or-vip>"
```

版本不要改：`v1.35.6-acp.1` / coredns `1.14.3-v4.4.6` / etcd `v3.6.13-acp.0`。`maxSurge: 0`。`skip-kube-proxy` 注解保留（kube-ovn）。

Kubernetes 1.35 若拉私有仓镜像失败，按官方升级文档在 kubelet patch JSON 里增加 `imagePullCredentialsVerificationPolicy`（本压模故意不写入该字段，现场按官方补丁加）。

```bash
grep -nE '<[^>]+>' global/control-plane/manifests/*.yaml
kubectl apply -f global/control-plane/manifests/02-control-plane-pool.yaml
kubectl apply -f global/control-plane/manifests/03-baremetal-cluster.yaml
kubectl apply -f global/control-plane/manifests/04-control-plane-machine-template.yaml
kubectl apply -f global/control-plane/manifests/05-cluster.yaml
kubectl apply -f global/control-plane/manifests/06-control-plane.yaml
```

首次交付不要 apply `global/worker/`。

## 12. 等待 Global 控制面 Ready

```bash
kubectl -n cpaas-system get baremetalcluster,cluster,kubeadmcontrolplane,machine,baremetalmachine
kubectl -n cpaas-system get events --sort-by=.lastTimestamp
kubectl -n cpaas-system get secret global-kubeconfig -o jsonpath='{.data.value}' | base64 -d > global-kubeconfig
kubectl --kubeconfig global-kubeconfig get nodes -o wide
```

期待：`BaremetalCluster/global` Ready；三台 Inventory 已分配；KCP `Ready=True`；Cluster `Phase: Provisioned`；三台 Master Node Ready。此时 **还不是** 拆 KIND 的时候。

## 13. 导入清单（必须在 POST installer **之前**）

缺这个 ConfigMap 时，handoff Job 目标列表为空，成品 Global 没有 BaremetalMachine / Inventory / Registration。

从 **已经 reconcile 的** BaremetalMachine 抄名字，不要用注册刚出来、尚未被 Machine 认领的列表：

```bash
kubectl -n cpaas-system get baremetalmachine \
  -l cluster.x-k8s.io/cluster-name=global \
  -o custom-columns='NAME:.metadata.name,INVENTORY:.status.machineInventoryRef.name'
kubectl -n cpaas-system get machineregistration.elemental.cattle.io
```

填进 `global/control-plane/bootstrap/04-dcs-import-extra-resources.yaml`。非 DR：不要加 encryption Secret 那一段（模板已省略）。不要导入 SeedImage 对象、Registration token Secret、plan Secret、KubeadmConfig。

```bash
kubectl apply -f global/control-plane/bootstrap/04-dcs-import-extra-resources.yaml
kubectl -n cpaas-system get cm dcs-import-extra-resources -o yaml
```

## 14. POST installer

文件：`global/control-plane/bootstrap/05-installer-config-baremetal.json`

改密码、VIP、`hostIP: 156.3.1.26`。`registry.domain` 必须是 `<platform-access-domain-or-vip>:11443`，**不要**写引导仓库。不要设 `registry.externalAddress`。`ha.isThirdParty: true`。产品保持 `base` + `acp`。

`console.host` 是控制面 VIP（不要带 `https://`、不要带端口）；平台入口写在 `console.globalHost`。

把改好的副本存成引导节点上的 `installer-config.json`（已 gitignore），**不要**把填了密码的文件提交。POST 前确认 JSON 合法、密码段已经是 base64：

```bash
python3 -m json.tool installer-config.json >/dev/null
# basic.password / registry.password 看起来应是字母数字 + =，不是你刚敲的明文
```

官方入口是 **installer Pod IP:8080**，不是平台 VIP，也不是 `https://156.3.1.26/cpaas-installer/...`。curl **必须在引导节点上跑**（Pod IP 只在 KIND CNI 里通，笔记本/跳板机到不了）：

```bash
kubectl -n cpaas-system get pods -l service_name=cpaas-installer -o wide
export INSTALLER_IP=$(kubectl get pods -n cpaas-system -l service_name=cpaas-installer \
  -o jsonpath='{.items[0].status.podIP}')
test -n "$INSTALLER_IP"
curl -sS -m 5 "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress" | head
curl -X POST "http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs" \
  -H 'Content-Type: application/json' \
  -d @installer-config.json
```

监控（通常 30–60 分钟）：

```bash
curl "http://${INSTALLER_IP}:8080/cpaas-installer/api/progress"
tail -f /var/cpaas/data/installer.log
kubectl --kubeconfig global-kubeconfig get nodes
kubectl --kubeconfig global-kubeconfig get clustermodule global
```

## 15. 交接门槛（现在还不能拆 KIND）

installer Success **不等于** 可以拆引导集群。官方要求同时满足：

```bash
kubectl --kubeconfig global-kubeconfig -n cpaas-system \
  get configmap baremetal-system-agent-handoff -o jsonpath='{.data.ready}{"\n"}'
# 必须是 true。不要手工 patch。

kubectl --kubeconfig global-kubeconfig -n cpaas-system \
  get machineinventories.elemental.cattle.io \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.plan.state}{"\n"}{end}'
# 每台导入的 Global 主机 status.plan.state = Applied

kubectl --kubeconfig global-kubeconfig -n cpaas-system \
  get rolebinding baremetal-global-system-agent-handoff-bridge
# 必须 NotFound
```

非 DR（本包 `splitAuthEnabled: false`）：成品 Global 只有共享身份 `baremetal-system-agent`。

## 15.1 确认仓库注解已经离开引导节点

官方文档：installer 把 **Cluster / DCSCluster** 的 `cpaas.io/registry-address` 改成平台仓库。Bare Metal 没有 DCSCluster；**BaremetalCluster 不在改写名单里**。拆 KIND 之前，**两边 kubeconfig 上**两份对象都必须指向平台仓库，否则节点还去打 `156.3.1.26:11443`。

```bash
# 引导集群（minialauda）—— installer 先改这里
kubectl -n cpaas-system get cluster,baremetalcluster global -o \
  custom-columns='KIND:.kind,REG:.metadata.annotations.cpaas\.io/registry-address'
# 成品 Global（导入之后才有）
kubectl --kubeconfig global-kubeconfig -n cpaas-system get cluster,baremetalcluster global -o \
  custom-columns='KIND:.kind,REG:.metadata.annotations.cpaas\.io/registry-address'
```

期待全部是 `<platform-access-domain-or-vip>:11443`。

- Cluster 仍是 `156.3.1.26:11443`：installer 没改写完，先别拆 KIND，先查 installer 日志。不要手工改 Cluster 就当交接成功。
- BaremetalCluster 仍是 `156.3.1.26:11443`、Cluster 已经是平台仓库：这是预期缺口，**patch 还停在引导 IP 的那一份**，不要 `kubectl apply -f 03-baremetal-cluster.yaml`（那份文件还写着引导 IP）：

```bash
# 哪边 get 出来还是 156.3.1.26，就在哪边 patch
kubectl -n cpaas-system annotate baremetalcluster global \
  cpaas.io/registry-address=<platform-access-domain-or-vip>:11443 --overwrite
# 或：
kubectl --kubeconfig global-kubeconfig -n cpaas-system annotate baremetalcluster global \
  cpaas.io/registry-address=<platform-access-domain-or-vip>:11443 --overwrite
```

不要把 API VIP 填进这个注解（VIP 是 6443，仓库是 11443）。

**现在先去做第 16–17 步。** 第 17 步完成之前，引导节点还要用来 `ctr push` OS 镜像，KIND 先留着。

---

# 第二部分：把 Global 变成管理集群

引导仓库里的 ISO / catalog 不能给 Workload 用。在 **成品 Global** 上重新推镜像和 chart。

## 16. 推 OS 和 provider 到平台仓库

导入 CRD **不会**在成品 Global 上安装 provider。Workload SeedImage 从 `ProductBase.spec.registry.address` 拉 ISO；installer 会把 **Cluster** 的 `cpaas.io/registry-address` 改成平台仓库（BaremetalCluster 见第 15.1 步）。引导仓库里的镜像 **不会**跟着过来，不推就会 `MANIFEST_UNKNOWN`。

在 **引导节点**上读成品仓库地址和密码（kubeconfig 指向 Global）：

```bash
export GLOBAL_KUBECONFIG=./global-kubeconfig
export OS_IMAGE_TAG=<os-image-tag>
GLOBAL_REGISTRY_ADDRESS="$(kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" \
  get productbase.product.alauda.io base -o jsonpath='{.spec.registry.address}')"
GLOBAL_REGISTRY_PASSWORD="$(kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" -n cpaas-system \
  get secret registry-admin -o jsonpath='{.data.password}' | base64 -d)"
echo "${GLOBAL_REGISTRY_ADDRESS}"   # 应类似 <platform-access-domain-or-vip>:11443
```

OS 镜像仍从引导节点的 `bare-metal-os` namespace 推（`--skip-verify` 必须带 `--local`）：

```bash
for repo in baremetal-base-image baremetal-base-image-iso; do
  ctr -n bare-metal-os images push --local --skip-verify \
    -u "admin:${GLOBAL_REGISTRY_PASSWORD}" \
    "${GLOBAL_REGISTRY_ADDRESS}/tkestack/${repo}:${OS_IMAGE_TAG}" "${repo}:${OS_IMAGE_TAG}"
  curl -sk -u "admin:${GLOBAL_REGISTRY_PASSWORD}" \
    "https://${GLOBAL_REGISTRY_ADDRESS}/v2/tkestack/${repo}/tags/list"
done
```

两条 tags/list 都必须含 `${OS_IMAGE_TAG}`。确认后再：

```bash
ctr -n bare-metal-os images rm \
  "baremetal-base-image:${OS_IMAGE_TAG}" \
  "baremetal-base-image-iso:${OS_IMAGE_TAG}"
```

两个 provider tgz 用 packtool 推到 **同一** `${GLOBAL_REGISTRY_ADDRESS}`。成品仓库有认证。用户是 `admin`，密码是上面的 `GLOBAL_REGISTRY_PASSWORD`（明文，不是 installer JSON 里那串 base64）。**先看 help**，不要假设 `-u`：

```bash
PACKTOOL=/root/cpaas-install/installer/res/amd64/packtool
$PACKTOOL pack push -h
# 按 help 选认证开关。下面只是一种常见形状：
$PACKTOOL pack push \
  -r "${GLOBAL_REGISTRY_ADDRESS}" \
  -u "admin:${GLOBAL_REGISTRY_PASSWORD}" \
  -c /root/cluster-api-provider-kubeadm.amd64.<kubeadm-provider-chart-version>.tgz
$PACKTOOL pack push \
  -r "${GLOBAL_REGISTRY_ADDRESS}" \
  -u "admin:${GLOBAL_REGISTRY_PASSWORD}" \
  -c /root/cluster-api-provider-baremetal.amd64.<baremetal-provider-chart-version>.tgz
```

help 里若是 `--username` / `--password` 就改用那对。推完再 apply `global/management/`，不要空仓 apply。

引导节点要能解析并访问 `${GLOBAL_REGISTRY_ADDRESS}`（HTTPS 11443）。`ctr --skip-verify` 只跳过证书校验，DNS/路由不通仍然会失败。

不要把引导集群那份带 IP SAN 的 `dex.tls` import 进成品 Global。

## 17. 在 Global 上安装两个 provider

先看成品 `dex.tls` 实际有哪些 key。installer `console.cert.selfSigned` 通常只有 `tls.crt,tls.key`，**没有 `ca.crt`**。压模 `global/management/02` 已把 `caCertSecretKey` 写成 `tls.crt`。若现场 Secret 有 `ca.crt`，把该文件改回 `ca.crt`。

```bash
# jq 可选。没有 jq 用 python3：
kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" -n cpaas-system \
  get secret dex.tls -o jsonpath='{.data}' | python3 -c 'import json,sys; print(",".join(json.load(sys.stdin).keys()))'
```

文件（**成品 Global**，不要 apply 到 minialauda）：

- `global/management/01-kubeadm-provider-apprelease.yaml`
- `global/management/02-baremetal-provider-apprelease.yaml`

改 chart 版本、`<platform-access-domain-or-vip>`、`<os-image-tag>`。`repoURL` 是平台仓库，不是 `172.18.0.1`。

```bash
kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" apply -f global/management/01-kubeadm-provider-apprelease.yaml
kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" apply -f global/management/02-baremetal-provider-apprelease.yaml
kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" -n cpaas-system get deploy \
  cluster-api-provider-baremetal-manager elemental-operator
```

chart 在 Global 上建的 catalog **是空的**，不会继承引导集群那条。`management/02` 的 `provider.imageCatalog.images` 一般会写入；仍空就 **merge patch**，不要 replace 整个 ConfigMap：

```bash
kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" -n cpaas-system \
  patch configmap elemental-image-catalog --type merge \
  -p "{\"data\":{\"v1.35.6-acp.1\":\"${GLOBAL_REGISTRY_ADDRESS}/tkestack/baremetal-base-image:${OS_IMAGE_TAG}\"}}"
kubectl --kubeconfig "${GLOBAL_KUBECONFIG}" -n cpaas-system \
  get configmap elemental-image-catalog -o jsonpath='{.data}'
```

`ImageCatalogMiss` 是终态：补 catalog 之后那个 BaremetalMachine 不会自己好，要删 Machine 让 KCP/MachineSet 重建。所以 **先有 catalog，再 apply Workload**。

### 拆 KIND（第 15 + 16 + 17 都过了才拆）

只删引导节点上的 minialauda / KIND 网络。不要 `kubectl delete cluster global`。拆完引导节点不再是 elemental `:12443` 入口。

官方没有给出 `kind delete` 的固定一行，现场用 Core Package 带的工具拆 **名叫 minialauda 的 KIND 集群**。先确认名字，再删：

```bash
# 现在 kubectl 仍指向引导集群时：
kubectl config current-context
kubectl get nodes
# 常见：kind / nerdctl 都能看到 minialauda
kind get clusters 2>/dev/null || true
nerdctl ps --format '{{.Names}}' | grep -i minialauda || true
docker ps --format '{{.Names}}' | grep -i minialauda || true
```

确认容器是 `minialauda-control-plane` 这类名字之后，**只删这个 KIND 集群**：

```bash
# 优先官方 KIND CLI（setup.sh 装过的话）
kind delete cluster --name minialauda
# 没有 kind 命令时，用 nerdctl/docker 停掉 minialauda 相关容器和 KIND 网络
# nerdctl rm -f minialauda-control-plane
# nerdctl network ls | grep kind
```

拆之前把 `global-kubeconfig` 拷离引导节点，之后所有 `kubectl` 都用它（或导入操作机）。拆完不要再对 minialauda apply 任何东西。

---

# 第三部分：Workload（在成品 Global 上）

`kubectl` 已连 **成品 Global**（`--kubeconfig global-kubeconfig`，或已经 `export KUBECONFIG=`）。KIND 拆掉之后没有 minialauda 可连。集群名全局替换 `<workload-cluster-name>`，**不能** 叫 `global`。

## 18. 控制面 Registration / ISO

文件：`workload/control-plane/manifests/01-registration-seedimage.yaml`

```text
baseImage: <platform-access-domain-or-vip>:11443/tkestack/baremetal-base-image-iso:<os-image-tag>
```

```bash
kubectl apply -f workload/control-plane/manifests/01-registration-seedimage.yaml
kubectl -n cpaas-system get sc
ISO_URL=""
for i in $(seq 1 60); do
  ISO_URL=$(kubectl -n cpaas-system get seedimage <workload-cluster-name>-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
curl -fL --retry 3 -o /root/<workload-cluster-name>-cp.iso "$ISO_URL"
# Registration URL 仍是凭证，只在终端看：
kubectl -n cpaas-system get machineregistration <workload-cluster-name>-registration \
  -o jsonpath='{.status.registrationURL}{"\n"}'
```

不要复用 Global 控制面 ISO，也不要复用 Workload 这张 ISO 去装 Worker。

Live ISO 软链、BMC disk-first、确认离开 Live：同第 7–8 步。无 DHCP 同样先配静态 IP，但目标改成平台域名 `:443` / 仓库 `:11443`，不再是引导节点 `:12443`。

## 19. Workload 控制面 CAPI

Inventory 填进 `02-control-plane-pool.yaml`。然后：

```bash
kubectl apply -f workload/control-plane/manifests/02-control-plane-pool.yaml
kubectl apply -f workload/control-plane/manifests/03-baremetal-cluster.yaml
kubectl apply -f workload/control-plane/manifests/04-control-plane-machine-template.yaml
kubectl apply -f workload/control-plane/manifests/05-cluster.yaml
kubectl apply -f workload/control-plane/manifests/06-control-plane.yaml
```

`03` External 时删 `vrid`。KCP 只改 ssh 公钥和 `<workload-control-plane-vip>` SAN。CIDR 不要和 Global / 其它 Workload 冲突。

```bash
kubectl -n cpaas-system get cluster,kubeadmcontrolplane,machine -l cpaas.io/cluster-name=<workload-cluster-name>
kubectl -n cpaas-system get secret <workload-cluster-name>-kubeconfig \
  -o jsonpath='{.data.value}' | base64 -d > workload-kubeconfig
kubectl --kubeconfig workload-kubeconfig get nodes -o wide
```

三台 Master Ready 后再做 Worker。

## 20. Workload Worker

文件：`workload/worker/manifests/01-registration-seedimage.yaml` 起一张 **新** ISO。软链流程同第 7 步。

```bash
kubectl apply -f workload/worker/manifests/01-registration-seedimage.yaml
ISO_URL=""
for i in $(seq 1 60); do
  ISO_URL=$(kubectl -n cpaas-system get seedimage <workload-cluster-name>-worker-registration-iso \
    -o jsonpath='{.status.downloadURL}' 2>/dev/null || true)
  [ -n "$ISO_URL" ] && break
  echo "waiting downloadURL ($i/60)..."
  sleep 30
done
test -n "$ISO_URL"
curl -fL --retry 3 -o /root/<workload-cluster-name>-worker.iso "$ISO_URL"
# 等 inventory Ready
kubectl apply -f workload/worker/manifests/02-worker-pool.yaml
kubectl apply -f workload/worker/manifests/03-worker-machine-template.yaml
kubectl apply -f workload/worker/manifests/04-worker-kubeadm-config-template.yaml
kubectl apply -f workload/worker/manifests/05-worker-machine-deployment.yaml
```

`replicas` 按现场改；`maxSurge` 必须是 `0`。

Worker 需要 VLAN 时：**Live ISO 里先配 NetworkManager，再等注册**。IP 配在 VLAN 口（如 `eth0.329`），不要配在 `eth0`。Pool 的 `networkDevice` 填接口名，不要填 connection 名。不要改 `BaremetalCluster.networkDevice` 来表达 VLAN（控制面仍走 `eth0`）。VLAN 地址不要占用 API VIP。先看 Inventory `observedNetwork` 再填 Pool。

```bash
# 地址 / VLAN id / 网卡名全部换成这台 Worker 的值
nmcli con add type vlan ifname eth0.329 con-name vlan329 dev eth0 id 329
nmcli con mod vlan329 \
  ipv4.method manual \
  ipv4.addresses <this-worker-ip>/<prefix> \
  ipv4.gateway <gateway> \
  ipv4.dns <dns> \
  ipv6.method ignore
nmcli con up vlan329
ping -c 3 <gateway>
ip -br addr show eth0.329
```

无 DHCP 的控制面同样：Live 里先有能访问平台仓库和 elemental 的地址，再软链。

---

# 排障与 FAQ

## ImageCatalogMiss

catalog 没有 `v1.35.6-acp.1`，或 tag 和 OCI 归档不一致。补 catalog 后 **删那个 Failed 的 Machine** 让控制器重建，不要删整个 Cluster。Workload 第一份 YAML 之前必须先有 catalog。

## elemental-operator 起不来（成品 Global）

日志：`references non-existent secret key: ca.crt`。installer 签发的 `dex.tls` 没有 `ca.crt`。`global/management/02` 已用 `tls.crt`；若你改回 `ca.crt` 就会再现。

## installer POST 连不上

URL 必须是 `http://${INSTALLER_IP}:8080/cpaas-installer/api/config/dcs`。不要 POST 到平台 VIP 或 `https://156.3.1.26/cpaas-installer/...`。curl 必须在引导节点上执行。先 `get pods -o wide` 确认 Pod IP，再 `curl -m 5` 打 progress。

## installer JSON 被拒 / 密码不对

`basic.password` 和 `registry.password` 是 **base64(明文密码)**，不是明文。编码：`printf '%s' '明文' | base64 | tr -d '\n'`。JSON 里不能写注释。用 `python3 -m json.tool installer-config.json` 先校验。

## SeedImage 一直不 Ready / downloadURL 为空

闸门是 `status.downloadURL` 非空，不要死等 `condition=Ready`（现场也见过 `SeedImageReady`）。先看：

```bash
kubectl -n cpaas-system get sc
kubectl -n cpaas-system get pvc
kubectl -n cpaas-system describe seedimage global-registration-iso
kubectl -n cpaas-system get pods | grep -i elemental
```

PVC `Pending` = 没有默认 StorageClass，20Gi 卷绑不上，ISO 永不构建。catalog 没有 `v1.35.6-acp.1`、`baseImage` 仓库地址错（引导阶段必须是 `172.18.0.1:11443`）、OS 镜像没推进仓库，同样会空。ISO 只能从能访问 `downloadURL` 的机器拉（通常就是引导节点）；笔记本直连多半 404。

## 拆 KIND 拆错了

只删名叫 `minialauda` 的 KIND 集群。不要 `kubectl delete cluster global`，也不要删成品 Global 的节点。拆前确认 `global-kubeconfig` 已经能 `get nodes`。

## 装完又进 Live ISO

BMC 仍从虚拟光驱启动。弹出 ISO，disk-first，再确认 `/` 不是 `LiveOS_rootfs`。

## `ln: failed` / 链到空路径

用了 `ls -s` 或盘符写错。`lsblk` 后再 `ln -sfn`。`test -b "$(readlink -f /dev/elemental-install-target)"`。

## 盘序对调

源用 `/dev/disk/by-id/wwn-*`，不要用 `/dev/sda`。

## Registration 连不上 elemental

两套 URL，不要混：

| 阶段 | elemental.server.url |
|---|---|
| 引导（minialauda） | `https://156.3.1.26:12443` |
| 成品 Global / Workload | `https://<platform-access-domain-or-vip>`（ingress **443**，没有 `:12443`） |

引导阶段 `dex.tls` 必须含引导节点 IP SAN。交接后新 Registration 走平台域名，不要再用引导 ISO。

## 节点还在打引导仓库 / MANIFEST_UNKNOWN 在 156.3.1.26

KIND 拆掉之后 `156.3.1.26:11443` 就是死地址。查：

```bash
kubectl -n cpaas-system get cluster,baremetalcluster -o \
  custom-columns='KIND:.kind,NAME:.metadata.name,REG:.metadata.annotations.cpaas\.io/registry-address'
```

Cluster 应由 installer 改成平台仓库。BaremetalCluster 若仍是引导 IP，按第 15.1 步 patch。Workload 对象如果写成了 `156.3.1.26`，改成平台仓库后让 Machine 重建，不要指望它自己好。不要把 API VIP 填进这个注解。

## 不要做的清理

```text
# 错误
kubectl delete cluster global
```

只拆 KIND minialauda，且必须在 OS/provider 已推到成品 Global、handoff Ready 之后。

## 已有操作系统的磁盘

Live ISO 安装会按 `install.device` 重分区。确认软链指向的就是要擦的系统盘。数据盘不要链上去。额外挂盘见官方 storage 文档，本包不放 storage YAML。

## HCS / 公有云不能 ISO 引导

不在本包范围。官方首装是 SeedImage ISO。qcow2 `--no-toolkit` 是研发旁路，不要和这套 4.4.1 ISO Global 混用。

---

# 文件对照（apply 前再扫一遍占位符）

JSON 不能写 `APPLY-ON`。installer 文件的目标写在 `global/control-plane/bootstrap/README.md`：引导节点 POST 到 installer **Pod IP:8080**，不是 kubectl apply。

**引导集群（minialauda）**

| 文件 | 必改 | 动作 |
|---|---|---|
| `global/control-plane/bootstrap/01-dex-tls-bootstrap.yaml` | 引导 IP 已填 `156.3.1.26` | apply |
| `global/control-plane/bootstrap/02-kubeadm-provider-apprelease.yaml` | chart 版本、platform host | apply |
| `global/control-plane/bootstrap/03-baremetal-provider-apprelease.yaml` | chart 版本、bootstrap IP、OS tag、VIP；elemental `:12443` | apply |
| `global/control-plane/manifests/01-registration-seedimage.yaml` | OS tag。闸门 `status.downloadURL` | apply |
| `global/control-plane/manifests/02-control-plane-pool.yaml` | 三台 inventory 名（对象 shape，不是官方 fragment） | apply |
| `global/control-plane/manifests/03-baremetal-cluster.yaml` | VIP；External 删 vrid。registry 先写引导 IP。**installer 之后不要再 apply** | apply 一次 |
| `global/control-plane/manifests/04-control-plane-machine-template.yaml` | 一般不用改 | apply |
| `global/control-plane/manifests/05-cluster.yaml` | 名必须是 `global`；仓库注解**第一次**写引导 IP。**installer 之后不要再 apply** | apply 一次 |
| `global/control-plane/manifests/06-control-plane.yaml` | ssh、etcd SAN。完整 KCP，不要用官方 fragment | apply |
| `global/control-plane/bootstrap/04-dcs-import-extra-resources.yaml` | 与 Pool 相同的 inventory 名；KCP Ready 后再填 | apply |
| `global/control-plane/bootstrap/05-installer-config-baremetal.json` | 密码（base64）、仓库、VIP、hostIP | **不是 apply**。副本 `installer-config.json`，POST Pod IP:8080 |

**成品 Global**

| 文件 | 必改 | 动作 |
|---|---|---|
| `global/management/01-kubeadm-provider-apprelease.yaml` | chart 版本、platform host | `--kubeconfig global-kubeconfig` apply |
| `global/management/02-baremetal-provider-apprelease.yaml` | chart 版本、platform host、OS tag；`caCertSecretKey: tls.crt`；elemental 走平台域名 **:443** | 同上 |

**Workload（成品 Global）**

| 文件 | 必改 |
|---|---|
| `workload/control-plane/manifests/03` 和 `05` | 集群名（不能是 `global`）；`cpaas.io/registry-address` **必须是平台仓库**，不要写 `156.3.1.26` |
| `workload/control-plane/manifests/*` | OS tag、inventory、VIP、ssh |
| `workload/worker/manifests/*` | 集群名、OS tag、inventory、ssh |

**不要 apply**

| 路径 | 原因 |
|---|---|
| `global/worker/manifests/` | 首次交付跳过。引导期 ISO，`baseImage` 是 `172.18.0.1:11443`。YAML 已戳 `DO-NOT-APPLY` |
| `global/control-plane/manifests/03` 和 `05`（installer 之后） | 会把 `cpaas.io/registry-address` 写回引导 IP |

交接后若加 Global Worker：`global/worker/post-global/01-registration-seedimage.yaml`（平台仓库 ISO）。
