---
title: kubernetes服务部署
categories: kubernetes
tags: kubernetes
---

#### 连接 harbor 仓库
```
# harbor账号密码全部正确，死活上不去，重新执行 harbor install 
# -n 命名空间
kubectl create secret docker-registry harbor --docker-server=192.168.1.142 --docker-username=admin --docker-password=Harbor12345 [-n 5g]

# 查询密码
kubectl get secret
```

#### 基本命令
```
# 保存docker镜像到tar文件
docker save <Image Name>:<TAG> -o <Image Name>_<TAG>.tag
# 加载镜像到docker
# docker load -f <Image Name>_<TAG>.tag

# 切换为minikube内部使用的docker
eval $(minikube docker-env)

# 查看pod
kubectl get pod -n ns-test
# 可以查看到 pod 部署在那台服务器上面
kubectl get pod -n ns-test -o 
kubectl get pod -n 5g pod-name -o yaml
# 查看pod启动日志
kubectl logs pod名称 -n ns-test

# 查看服务
kubectl get svc -n ns-test
kubectl describe svc nginx-service -n ns-test
# 删除service
kubectl delete service 服务名称 -n 工作空间名称

# 查看k8s所有的资源命令
kubectl api-resources

# 根据yaml文件强制替换原有的资源
kubectl replace --force -f yaml文件
```

#### node相关
```
kukectl get node 
kukectl describe node node名称
```

#### apiVersion kind相关
```
# 查看k8s所有的api版本
kubectl api-versions
# 查看所有的资源
# --namespaced=true 该资源是否有指定的namespace
kubectl api-resources [-o wide] [--namespaced=true]
# 查看kind对应的api版本
kubectl explain pod.apiVersion
# 查看资源清单格式
kubectl explain pod
```

#### 服务编排流程

##### 1. 创建工作空间
```
# namespace yaml文件内容
apiVersion: v1  
kind: Namespace  #类型为Namespace
metadata:
  name: ns-test  #namespace 名称
  labels:
    name: label-test    #namespace标签
    
# 创建
kubectl create -f namespace.yml
kubectl get namespace
kubectl describe namespace 命名空间名称
kubectl delete namespace 命名空间名称
```

##### 2. 使用deploy部署服务
```
# pod yaml 文件内容
apiVersion: apps/v1 # apps不能省略
kind: Deployment
metadata:
  namespace: ns-test
  name: nginx-pod
  
spec:
  selector:
    matchLabels:
      app: nginx
  # 集群数量
  replicas: 1
  #设置滚动升级策略
  #Kubernetes在等待设置的时间后才开始进行升级，例如5s
  minReadySeconds: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      #在升级过程中最多可以比原先设置多出的Pod数量
      maxSurge: 1
      #在升级过程中Deployment控制器最多可以删除多少个旧Pod，主要用于提供缓冲时间
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector: # 设置节点选择器
        node-label: node-label-value # 节点的标签和值全部匹配
      # imagePullSecrets 的账号需要和当前的namespace一致
      # 拉取镜像的密码，通过该kubectl create secret配置
      # 如果是http的镜像地址，kubectl create secret设置的无效，直接使用docker login登录，该处配置注释掉
      imagePullSecrets:
      - name: harbor
      serviceAccountName: admin-user
      containers: 
      - name: nginx
        image: registry.cn-hangzhou.aliyuncs.com/inus/k8s:nginx-latest
        resources:
          requests: # 初始化申请的资源
            memory: "1Gi"
            cpu: "200m"
          limits: # 限制的最大资源
            memory: "2Gi"
            cpu: "400m"
        ports: 
        - containerPort: 80

# 创建
kubectl create -f nginx-pod.yml
kubectl get pod -n ns-test
kubectl describe pod pod名称 -n 命名空间名称
kubectl delete pod pod名称 -n 命名空间名称
```

##### 3.创建service
```
# service yaml 文件内容
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: ns-test
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
  - protocol: TCP
    #Service在集群中暴露的端口（用于Kubernetes服务间的访问）
    port: 80
    #Pod上的端口（与制作容器时暴露的端口一致，在微服务工程代码中指定的端口）
    targetPort: 80
    #K8s集群外部访问的端口（外部机器访问）
    nodePort: 30002
    
# 创建
kubectl create -f nginx-service.yml
kubectl get service -n ns-test
kubectl describe service service名称 -n ns-test
kubectl delete service service名称 -n ns-test
```

#### 部署问题

##### 0/1 nodes are available: 1 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.

```
# 删除污点
kubectl taint nodes --all node-role.kubernetes.io/master-

# 如果不允许调度
# 污点可选参数
    NoSchedule: 一定不能被调度
    PreferNoSchedule: 尽量不要调度
    NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod
kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule
```

#### 无法删除namespace，命名空间一直Terminating
> 直接调用接口删除namespace

```
# 启动本地代理
kubectl proxy --address=0.0.0.0 --port=8001 --accept-hosts='^*$'

# 导出namespace的配置文件
kubecctl get ns 命名空间名称 -o json > tmp.json

# 修改finalizers为[]
cat tmp.json | grep finalizers

# 调用接口删除namespace
curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://192.168.1.142:8001/api/v1/namespaces/命名空间名称/finalize
```
