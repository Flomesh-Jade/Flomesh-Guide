Flomesh服务网格部署手册
=== 
# 1.资源准备
- 准备一个K8S集群（1.21版本以上），支持K3S等其他方式快速构建K8S集群，本手册使用k3s构建集群作参考
- 安装helm工具（3.10版本以上）
- 准备安装物料
```bash
git clone https://github.com/Flomesh-Jade/Flomesh-Guide.git
cd 
```

# 2.安装flomesh-gui
安装clickhouse
> clickhouse用作日志服务器，收集sidecar日志信息，提供给flomesh-gui进行整合展示
```bash
kubectl create ns flomesh-gui
kubectl apply -f -n flomesh-gui ./yaml/flomesh-gui/clickhouse.yaml
kubectl wait --for=condition=ready pod -n flomesh-gui -l app=clickhouse --timeout=180s
```
安装postgres
> postgres用作系统数据库，保存flomesh-gui系统配置
```bash
kubectl apply -f -n flomesh-gui ./yaml/flomesh-gui/postgres.yaml
kubectl wait --for=condition=ready pod -n flomesh-gui -l app=postgres --timeout=180s
```
安装pipy-repo
> pipy-repo在FLB场景下用作系统pipy codebase库，是flomesh-gui必须组件
```bash
kubectl apply -f -n flomesh-gui ./yaml/flomesh-gui/pipy-repo.yaml
kubectl wait --for=condition=ready pod -n flomesh-gui -l app=pipy-repo --timeout=180s
```
安装flomesh-gui
> flomesh-gui提供图形化配置和展示页面
- 下载flomesh-gui镜像并导入（以k3s为例）
```bash 
export version="2.0.0-199"
wget http://repo.flomesh.cn/images/flomesh-gui-alpine-$version.tar.gz
gunzip flomesh-gui-alpine-$version.tar.gz
k3s ctr image import flomesh-gui-alpine-$version.tar
```
- 部署flomesh-gui
```bash
kubectl apply -f -n flomesh-gui ./yaml/flomesh-gui/flomesh-gui.yaml
kubectl wait --for=condition=ready pod -n flomesh-gui -l app=flomesh-gui --timeout=180s
```
# 3.安装FSM
- 下载fsm chart包
```bash
wget https://github.com/flomesh-io/fsm-classic/raw/gh-pages/fsm-0.2.6.tgz
```
- 编辑values.yaml
```bash
vi ./yaml/fsm/values.yaml
```
```yaml
# 修改ingress.namespaced=true
  ingress:
    name: fsm-ingress-pipy
    className: "pipy"
    enabled: true
    namespaced: true
    http:
      enabled: true
      port: 80
      containerPort: 8000
      nodePort: 30508
    tls :
      enabled: false
      port: 443
      containerPort: 8443
      nodePort: 30607
      mTLS: false
      sslPassthrough:
        enabled: false
        upstreamPort: 443
```
```yaml
# 修改flb配置
  flb:
    enabled: true                #开启flb后，支持提供k8s loadbalacer能力
    strictMode: false
    secretName: fsm-flb-secret
    baseUrl: http://<ip>:<port>  #flomesh-gui访问地址
    username: admin              #flomesh-gui用户
    password: Flomesh@2023       #flomesh-gui用户密码
    defaultCluster: cluster1     #flb集群
    defaultAddressPool: ap1      #flb地址池
    defaultAlgo: rr
```
- 部署fsm
```
helm install --namespace flomesh --create-namespace --set fsm.version=0.2.6 --set fsm.logLevel=5 -f ./yaml/fsm/values.yaml fsm ./charts/fsm-0.2.6.tgz
kubectl wait --for=condition=ready pod -n flomesh -l helm.sh/chart=fsm-0.2.6 --timeout=180s
```
# 4.安装OSM
- 下载并安装osm cli工具
```
system=$(uname -s | tr [:upper:] [:lower:])
arch=$(dpkg --print-architecture)
release=v1.3.6
curl -L https://github.com/cybwan/fsm/releases/download/${release}/fsm-${release}-${system}-${arch}.tar.gz | tar -vxzf -
./${system}-${arch}/fsm version
cp ./${system}-${arch}/fsm /usr/local/bin/
```
- 安装osm
```
export osm_namespace=osm-system
export osm_mesh_name=osm
osm install \
    --mesh-name "$osm_mesh_name" \
    --osm-namespace "$osm_namespace" \
    --set=osm.image.registry=flomesh \
    --set=osm.image.tag=1.3.6 \
    --set=osm.sidecarLogLevel=warn \
    --set=osm.controllerLogLevel=warn \
    --set=osm.remoteLogging.enable=true \
    --set=osm.remoteLogging.address=192.168.68.11 \
    --set=osm.remoteLogging.port=30023 \
    --set=osm.remoteLogging.authorization="Basic ZmxvbWVzaDpGbG9tZXNoQDEyMyE=" \
    --set=osm.featureFlags.enablePluginPolicy=true \
    --timeout=900s
```
> 这三条为clickhouse的接口地址、账户密码，密码采用base64形式
>  --set=osm.remoteLogging.address=192.168.68.11 \
    --set=osm.remoteLogging.port=30023 \
    --set=osm.remoteLogging.authorization="Basic ZmxvbWVzaDpGbG9tZXNoQDEyMyE=" \
