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
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml

### 3.1 以NodePort的方式部署ingress
如果以NodePort的方式部署ingress，需要apply 下面的文件
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml

文件内容是
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

### 3.1 以LoadBalancer的方式部署ingress
如果以NodePort的方式部署ingress，需要apply 下面的文件
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

自己指定了 externalIPs，这样就可以像minikube一样使用ingress了。 这种方式实际上是暴露了一个Node的ip，这种方式就不像第一种方式需要指定特定的端口，直接访问80或者443就可以了。

## 4. create Ingress

先使用
```
kubectl get pod nginx-ingress-controller-7995bd9c47-flxrm -n ingress-nginx -o wide
```
得到控制器分配在哪个node上。在我这个环境中被分配在enzyme1.fyre.ibm.com这个Node上

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

使用
```
kubectl get svc -n ingress-nginx
```
得到暴露出来的端口。

| NAMESPACE | NAME | TYPE | CLUSTER-IP | EXTERNAL-IP | PORT(S) | AGE |
| --- | --- | --- | --- | --- | --- | --- |
| ingress-nginx | ingress-nginx | NodePort | 10.101.124.233 | <none> | 80:32041/TCP,443:32283/TCP | 7h15m |

这样使用curl enzyme1.fyre.ibm.com:32041访问时，就能够获取到相应的pod的hostname

## 5.Service原理
