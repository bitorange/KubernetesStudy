# Create a operator


# 1. Install Go environment in a machine
- 从https://golang.google.cn/dl/ 下载相应的Go语言版本， 我下载了go1.13.linux-amd64.tar.gz
- tar -C /usr/local -xzf go1.13.linux-amd64.tar.gz
- 修改/etc/profile 文件, GOPATH是机器上将要存放你自己开发的Go project的路径
 ```
export PATH=$PATH:/usr/local/go/bin
export GOPATH=/opt/go
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOBIN
export GO111MODULE=on
 ```  
- 运行```go version ```检查go安装是否成功


# 2. 安装operator SDK
- 安装dep
```
curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
sudo yum install  build-dep  gcc
```
- 安装operator SDK
```
go get -d github.com/operator-framework/operator-sdk 
cd $GOPATH/src/github.com/operator-framework/operator-sdk
git checkout master
make tidy
make install
```

notes: 当执行make tidy时，可能会出错，ignore，继续执行下一步即可
- 运行```operator-sdk version```检查是否安装成功

# 3. 新建一个operator

## 3.1 初始化一个operator project
运行命令
```
operator-sdk new memcached-operator --repo=github.com/bitorange/memcached-operator
```
你会在$GOPATH/src下看到一个memcached-operator
```
cd $GOPATH/src/memcached-operator
```
其中cmd/manager/main.go是整个operator的入口。

## 3.2 添加一个 Custom Resource Definition
```
operator-sdk add api --api-version=cache.example.com/v1alpha1 --kind=Memcached
```
你可以在pkg/apis/cache/v1alpha1/memcached_types.go文件中添加你的spec 和 status。
```
type MemcachedSpec struct {
	// Size is the size of the memcached deployment
	Size int32 `json:"size"`
}
type MemcachedStatus struct {
	// Nodes are the names of the memcached pods
	Nodes []string `json:"nodes"`
}
```

每当你修改了 *_types.go文件之后你都需要运行```operator-sdk generate k8s```使得修改生效



## 3.3 添加一个新的controller
```
operator-sdk add controller --api-version=cache.example.com/v1alpha1 --kind=Memcached
```
你会看到一个新的文件夹pkg/controller/memcached。 修改pkg/controller/memcached/memcached_controller.go文件的内容去实现你的逻辑。 在memcached_controller.go文件中有几个地方需要特别关注。

- 监控改变

在memcached_controller.go文件中会监控集群中kind为Memcached的修改。 首先监控了Memcached type作为primary resource，对于Add/Update/Delete的事件，reconcile循环会为了Memcached对象发送一个reconcile的“请求(a namespace/name key)”。
```
err := c.Watch(
  &source.Kind{Type: &cachev1alpha1.Memcached{}}, &handler.EnqueueRequestForObject{})
```
在使用的例子中，将Deployment作为第二监控对象，你可以将Deployment换做你想要监控的资源。The next watch is for Deployments but the event handler will map each event to a reconcile Request for the owner of the Deployment. 
```
err := c.Watch(&source.Kind{Type: &appsv1.Deployment{}}, &handler.EnqueueRequestForOwner{
    IsController: true,
    OwnerType:    &cachev1alpha1.Memcached{},
  })
```
- Reconcile loop
每个controller都有一个Reconciler对象，在Reconcile()函数中实现了reconcile的循环。为reconcile的循环传递了一个参数 Request，这个结构体中有个参数NamespacedName中保存着namespace和name，这namespace/name可以作为key去cache中查询primary resource对象，这个对象在本例子中是Memcached。 解释一下，这里的cache就是本地维护的etcd数据库中的key:value键值对，通过key可以直接获取value，具体参见ApiServer的工作原理。


## 3.4 Build and run the operator

在部署之前需要先将crd注册到你的集群的Kubernetes apiserver中。
```
kubectl create -f deploy/crds/cache.example.com_memcacheds_crd.yaml
```
有两种方式去运行operator
- As a Deployment inside a Kubernetes cluster
- As Go program outside a cluster

### 3.4.1 Run as a Deployment inside the cluster
```operator-sdk build```命令会调用```docker build```命令，去生成一个operator的image。

在生成operator的image之前，如果在你的项目的目录下存在go.mod文件，运行命令
```
go mod vendor
```
接下来生成image并push到公共的repository，比如quay.io中
```
operator-sdk build quay.io/example/memcached-operator:v0.0.1
sed -i 's|REPLACE_IMAGE|quay.io/example/memcached-operator:v0.0.1|g' deploy/operator.yaml
docker push quay.io/example/memcached-operator:v0.0.1
```

以生成image的方式，你可以在任意集群下载 operator的image并部署，如果在其它的集群，需要拷贝deploy下面的文件到集群中。在image被生成之后，kubectl apply命令运行deploy文件夹下的四个文件，这四个文件设置了RBAC并部署operator。

```
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml
```
使用下面的命令保证memcached-operator启动成功并且在运行：
```
kubectl get deployment
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           1m
```

### 3.4.2 Run locally outside the cluster
你可以在开发和测试过程中使用这种部署方式。
先将OPERATOR_NAME设置到环境变量中：
```
export OPERATOR_NAME=memcached-operator
```
本地运行operator，有两种方法，一是使用本地kubernetes集群，二是使用其它的集群。

使用本地集群运行命令：
```
operator-sdk up local --namespace=default
```
如果使用其它集群需要在上面的命令之后加上```--kubeconfig=<path/to/kubeconfig>```


## 4. 创建一个Memcached的CR
根据你在3.2中的操作，当时创建了一个crd，并在memcached_types.go文件中添加了spc的值，所以可以创建一个Memcached的CR的文件cache.example.com_v1alpha1_memcached_cr.yaml。其实这个文件在deploy文件夹下也被生成了。文件内容为：

```
apiVersion: "cache.example.com/v1alpha1"
kind: "Memcached"
metadata:
  name: "example-memcached"
spec:
  size: 3
```
运行命令
```
kubectl apply -f deploy/crds/cache.example.com_v1alpha1_memcached_cr.yaml
```

使用命令去确认deploment部署成功
```
kubectl get deployment

NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
memcached-operator       1         1         1            1           2m
example-memcached        3         3         3            3           1m
```

使用命令去确认pod启动成功，并检查pod状态
```
kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
example-memcached-6fd7c98d8-7dqdr     1/1       Running   0          1m
example-memcached-6fd7c98d8-g5k7v     1/1       Running   0          1m
example-memcached-6fd7c98d8-m7vn7     1/1       Running   0          1m
memcached-operator-7cc7cfdf86-vvjqk   1/1       Running   0          2m
```
```
kubectl get memcached/example-memcached -o yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  clusterName: ""
  creationTimestamp: 2018-03-31T22:51:08Z
  generation: 0
  name: example-memcached
  namespace: default
  resourceVersion: "245453"
  selfLink: /apis/cache.example.com/v1alpha1/namespaces/default/memcacheds/example-memcached
  uid: 0026cc97-3536-11e8-bd83-0800274106a1
spec:
  size: 3
status:
  nodes:
  - example-memcached-6fd7c98d8-7dqdr
  - example-memcached-6fd7c98d8-g5k7v
  - example-memcached-6fd7c98d8-m7vn7
```
可以注意到这里的status是返回的node的nname，这里与3.2中向memcached_types.go文件中添加的Status的值有关。