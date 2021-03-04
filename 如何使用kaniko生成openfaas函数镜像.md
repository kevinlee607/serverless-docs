# 如何使用kaniko生成openfaas函数镜像

项目地址：https://github.com/openfaas/faas

文档地址： https://docs.openfaas.com/

**假定条件：**

- dockerhub: dockerhub.qingcloud.com
- kubernetes 空间： openfaas

## 下载模板

```
git clone https://github.com/openfaas/templates.git
```

## 使用示例

 以golang为例，说明如何通过kaniko生成镜像。

可以使用faas-cli生成函数开发脚手架

项目地址：https://github.com/openfaas/faas-cli

```
faas-cli new salary --lang go  
```



### 写函数

写函数的一些规范(golang为例，**非官方提供的模板有自己的规范**)：

- 函数名必须为handler
- 最好将vendor目录（golang dependency）一起提供
- 写函数前要明确函数的输入输出.

golang示例如下：

```go
package function
import (
	"encoding/json"
	"fmt"
)

type reqStr struct {
	UserName string  `json:"username"`
	Salary   float64 `json:"salary"`
	TaxRate  float64 `json:"taxrate"`
}

// Handle a serverless request
func Handle(req []byte) string {
	var userInfo reqStr
	err := json.Unmarshal(req, &userInfo)
	if err != nil {
		return fmt.Sprintln("wrong fromat was provided, error:", err.Error())
	}
	taxHomePay := userInfo.Salary - (userInfo.Salary * userInfo.TaxRate)
	return fmt.Sprintf("Hello, %s. 你的税后工资为: %f", string(userInfo.UserName), taxHomePay)
}
```

### 打包函数

workaroud： 将template里的go目录拷到salary目录下，并将写好的函数文件移动到go目录的function目录下，将vendor内容放到go目录的根目录下，如果已经有了，请将vendor内容拷贝到已有的vendor目录下。

然后将在个go的根目录下打包成tar,**注意：不要把go的目录打进tar包，只是go目录下的内容**

```
cd go
tar zcf ./salary.tar.gz ./*
```

上传tar包到qingstor

比如地址为：http://gd2.qingstor.com/openfaas/salary.tar.gz

## 配置kaniko做镜像的yaml文件

首先需要几个配置，docker的cofnig.json， qingstor的config.yaml，使用kubernetes的话需要使用创建这几个密钥或者配置,本文以docker是configmap，qinstor是secret为例。假设：docker配置文件/root/.docker/config.json，qingstor配置文件/root/.qingstor/config.yaml

```
kubectl create secret generic qingstor --from-file /root/.qingstor/config.yaml -n openfaas
kubectl create configmap docker-config -n openfaas --from-file=/root/.docker/config.json
```

qinstor的下载解压，我做了一个镜像用来初始化kaniko容器，dockerhub.qingcloud.com/openfaas/qsexec:v0.0.4，源码地址：

下载kaniko镜像，**不要自己制作kaniko镜像，这个浪费了我很长时间，用官方镜像**，镜像地址：gcr.io/kaniko-project/executor:latest，国内可能下载不了，可以使用梯子，下载然后同save --》 import到国内自己的私有镜像库里。例子如下：

```
ssh root@<墙外docker主机>
# 下拉镜像
docker pull gcr.io/kaniko-project/executor:latest
docker save gcr.io/kaniko-project/executor:latest -o executor.tar
# 本地主机
scp -rp root@<墙外docker主机>:/path/executor.tar .
docker import excutor.tar gcr.io/kaniko-project/executor:latest
docker tag gcr.io/kaniko-project/executor:latest dockerhub.qingcloud.com/openfaas/executor:latest
docker push dockerhub.qingcloud.com/openfaas/executor:latest  ## 前提请登录private registory。
```

编写yaml文件，内容如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  initContainers:
    - name: qsexec
      image: dockerhub.qingcloud.com/openfaas/qsexec:v0.0.4
      env:
        - name: BUCKET_NAME
          value: "openfaas"
        - name: QINGSTOR_REGION
          value: "gd2"
        - name: FILE_NAME
          value: "salary.tar.gz"
        - name: WORKSPACE
          value: "/repo"
      volumeMounts:
        - name: workspace
          mountPath: /repo
        - name: qingstor
          mountPath: /.qingstor/
  containers:
    - name: kaniko
      image: dockerhub.qingcloud.com/openfaas/executor:v0.1
      args:
        - "--dockerfile=/repo/Dockerfile"
        - "--context=dir:///repo"
        - "--destination=dockerhub.qingcloud.com/openfaas/test:v0.1"
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker/
        - name: workspace
          mountPath: /repo
  volumes:
  - name: docker-config
    configMap:
      name: docker-config
  - name: workspace
    emptyDir: {}
  - name:  qingstor
    secret:
      secretName: qingstor
  restartPolicy: Never
  imagePullSecrets:
    - name: qingcloud-openfaas
```

执行创建

```
kubectl create -f kaniko.yaml -n openfaas
```

部署函数 k8s参考：https://docs.openfaas.com/deployment/kubernetes/#use-a-private-registry-with-kubernetes

对于私有镜像仓库，需要创建私有仓库的密钥。

```
kubectl create secret docker-registry dockerhub \
    -n openfaas-fn \
    --docker-server=$DOCKER_SERVER_URL \
    --docker-username=$DOCKER_USERNAME \
    --docker-password=$DOCKER_PASSWORD \
    --docker-email=$DOCKER_EMAIL
```

注意函数部署时需要此仓库密钥名称：

```
secrets:
      - dockerhub
```

## 部署函数

### 通过ui部署

访问openfaas UI地址如： http://gateway.clouden.io:31068/ui,   **注意此UI地址根据实际情况进行修改。**

在部署页面中主要修改以下参数：

```
Docker image： dockerhub.qingcloud.com/openfaas/test:v0.1   # 此处填写docker image的镜像地址。 必填
Function name: testinfo                                     # 此处填写函数名称，需faas内全局唯一。必填
Function process： testinfo                                 # 非必填，此处填写faas的主执行函数名称，建议不填写。选填
Network:                                                    # 为docker swarm参数，k8s不需要。选填
Environment Variables:                                # 系统变量。选填。classic-watchdog 参考：
                                                      # https://github.com/openfaas/workshop/blob/master/lab8.md
                                                      # of-watch:https://github.com/alexellis/go-long/blob/master/stack.yml
Secrect:  dockerhub                                   # 使用的k8s的密钥。私有docker仓库必填。
Labels：                                              # 开业参考：https://docs.openfaas.com/architecture/autoscaling/
```

### 如何请求函数

使用post访问函数。地址为：http://gateway.clouden.io:31068/function/testinfo

使用示例如下：

```
curl -X POST -d '{"username":"test","salary":5000, "taxrate":0.15}' http://gateway.clouden.io:31068/function/testinfo1
```

