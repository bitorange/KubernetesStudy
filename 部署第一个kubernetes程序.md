# Deployment, Service and Ingress
## 1. create a deployment for nginx
deployment的文件内容 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
```
执行命令

```
kubectl apply -f deployment.yaml
```
执行成功之后，可以使用下面的命令查看结果

```
kubectl get deployment
kubectl get pods
```

## 2. create Service
service的文件内容
```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376

``` 

## 3. enable nginx Ingress
你如果想要在你的集群中使用ingress，你需要安装ingress的controller，并建立相应的service。 ingress的controller是在集群中，那么怎么使得外部能够访问Ingress controller，从而使用集群的服务呢？ 有三种方式，下面我将介绍其中的两种，有一种需要用到云服务商的loadBalancer，我就不在这里介绍了。

### 3.1 以NodePort的方式部署ingress

这种方式先以Deployment的方式部署ingress的controller，然后再给它添加一个Type是NodePort的Service。首先需要apply 下面的文件去部署ingress controller的Deployment，顺便还会建立一个ingress-nginx的namespace。 这个文件的地址是：

https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml

然后部署一个Service，部署service的文件内容是
```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```
在两个文件都被kubectl apply之后，使用 ``` kubectl get svc -n ingress-nginx ``` 可以得到 80:3xxxx/TCP和443:3yyyy/TCP格式的端口号，这个端口号在每个节点都能访问到。 那么在你部署了ingress关联到你的service之后，你就可以访问 任意集群节点ip + 3xxxx 去访问你的服务了。

### 3.1 以LoadBalancer的方式部署ingress
这种方式和第一种方式差不多，只不过在部署service的时候以LoadBalancer的方式去部署。先部署ingress的Deployment（参考3.1），然后部署service，需要apply 下面的文件
```
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
  externalIPs:
    - 9.30.198.171    
```

这种方式的特点是自己指定了externalIPs，我指定的externalIPs是我集群的master节点的IP，这样就可以像minikube一样使用ingress了，你可以在你的局域网中直接访问ip地址了。 

这种方式实际上是暴露了一个Node的ip，这种方式就不像第一种方式需要指定特定的端口，直接访问80或者443就可以了。你也可以不指定externalIPs，但是不指定ip就需要你的集群是部署在云服务提供商的平台上，自建的集群一般是没有的。

## 4. create Ingress

下面你就可以创建ingress的object去关联你的service使得你的服务能够在外部访问了。下面文件中，enzyme1.fyre.ibm.com是9.30.198.171机器的hostname，他们指向同一个节点。

ingress的文件内容
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: enzyme1.fyre.ibm.com
    http:
      paths:
      - path:
        backend:
          serviceName: hostnames
          servicePort: 80
``` 

如果你在集群中使用NodePort的方式部署你的ingress service，那么你需要使用 enzyme1.fyre.ibm.com:3xxxx 去访问你的pod，这样你就能够获取到相应的pod的hostname。

如果你在集群中使用LoadBalancer的方式部署你的ingress service，那么你需要使用 enzyme1.fyre.ibm.com:80,或者 enzyme1.fyre.ibm.com:443 去访问你的pod。
