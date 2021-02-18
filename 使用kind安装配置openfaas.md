# 使用kind安装配置openfaas

本文是使用manjaro系统为基础写的文档。

## 安装kind

kind依赖docker或者podman，docker按照相对比较简单。安装docker

```
sudo pacman -S docker
```

docker其他安装请参考：https://docs.docker.com/engine/install/

安装kind

```
sudo pacman -S kind
```

kind源码安装请参考 https://kind.sigs.k8s.io/docs/user/quick-start/#installation

## 创建cluster

因为不想使用docker hub，所以本文使用本地docker registry作为仓库。

使用脚本创建cluster，内容如下：

```
#!/bin/sh
set -o errexit

# create registry container unless it already exists
reg_name='kind-registry'
reg_port='5000'
running="$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)"
if [ "${running}" != 'true' ]; then
  docker run \
    -d --restart=always -p "${reg_port}:5000" --name "${reg_name}" \
    registry:2
fi

# create a cluster with the local registry enabled in containerd
cat <<EOF | kind create cluster --name openfaas --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:${reg_port}"]
    endpoint = ["http://${reg_name}:${reg_port}"]
EOF

# connect the registry to the cluster network
docker network connect "kind" "${reg_name}"

# tell https://tilt.dev to use the registry
# https://docs.tilt.dev/choosing_clusters.html#discovering-the-registry
for node in $(kind get nodes); do
  kubectl annotate node "${node}" "kind.x-k8s.io/registry=localhost:${reg_port}";
done
```

## 安装metallb作为负责均衡管理器

### 创建metallb-system的namespace

创建文件namespace.yml

```
apiVersion: v1
kind: Namespace
metadata:
  name: metallb-system
  labels:
    app: metallb
```

创建namespace

```
kubectl apply -f ./namespace.yml
```

### 安装metallb

创建metallb secrets

```
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)" 
```

下一步安装可能会失败，因为国内访问被墙的问题，可能通过翻墙下载下来，在进行安装。

```
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.1/manifests/metallb.yaml
```

另一种方式，是下载metallb的源码安装

```
git clone https://github.com/metallb/metallb.git
cd metallb
sudo kubectl apply -f manifests/metallb.yaml
```

### 配置lb的地址池

获取可用地址

```
sudo docker network inspect -f '{{.IPAM.Config}}' kind

[{172.18.0.0/16  172.18.0.1 map[]} {fc00:f853:ccd:e793::/64  fc00:f853:ccd:e793::1 map[]}]
```

创建配置文件metallb-configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.18.255.200-172.18.255.250
```

使配置生效：

```
sudo kubectl apply -f ./metallb-configmap.yaml
```

## 安装openfaas服务

从github上下载源码

````
git clone https://github.com/openfaas/faas-netes.git
````

创建openfaas服务组件namespace

```
cd faas-nets
[kevin@kevin-pc faas-netes]$ sudo kubectl apply -f namespaces.yml 
[sudo] password for kevin: 
namespace/openfaas created
namespace/openfaas-fn created
```

创建openfaas登录用户名和密码

```
sudo kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password=ZHU88jie
```

安装openfaas服务组件

```
[kevin@kevin-pc faas-netes]$ sudo kubectl apply -f ./yaml/
```

验证是否安装成功,所有pod都为running即为安装成功

```
sudo kubectl -n openfaas get pods -watch
```

### 修改service使用LoadBalancer作为访问入口

创建LB的service配置文件 gateway-loadbalancer.yaml

```
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: openfaas
    component: gateway
  name: gateway-external
  namespace: "openfaas"
spec:
  type: LoadBalancer
  ports:
    - name: gateay-lb
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: gateway
```

使service的配置生效

```
sudo kubectl apply -f ./gateway-loadbalancer.yaml
```

查看LB的IP地址

```
[kevin@kevin-pc yaml]$ sudo kubectl -n openfaas get svc gateway-external
[sudo] password for kevin: 
NAME               TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
gateway-external   LoadBalancer   10.96.17.212   172.18.255.200   8080:32493/TCP   123m
```

EXTERNAL-IP为LB的地址，访问地址 http://172.18.255.200:8080即可打开网页，输入用户名和密码即可登录。