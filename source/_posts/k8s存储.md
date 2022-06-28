---
title: kubernetes存储
categories: kubernetes
tags: kubernetes
---

#### 存储种类
![存储种类](http://inus-markdown.oss-cn-beijing.aliyuncs.com/img/image-20220615164455901.png)

##### 1. 临时存储
充当临时空间使用，pod中产生的数据不需要做持久化存储时可使用该存储
```
kind: pod
spec: 
  volumes:
  - name: volumn-name #卷名称
    emptyDir: {} #指定存储的方式为临时存储
  containers:
    volumeMounts:
    - name: volumn-name # 和上面的对应
      mountPath: /usr/share/nginx/html #挂载到容器中的路径
```

##### 2. 半持久化存储
hostpath，即与pod所运行的宿主机上面共享一个存数据的目录。\
当pod漂移到其他node上面时将会读取不到原宿主机上面的内容.
```
kind: pod
spec:
  volumes:
  - name: nginx-path
    hostPath:
      path: /opt/nginx/data #宿主机的目录
      type: DirectoryOrCreate #给定的目录不存在则创建
  containers:
    volumeMounts:
    - name: nginx-path # 和上面的对应
      mountPath: /usr/share/nginx/html #挂载到容器中的路径
```

hostPath.type | 说明
---|---
DirectoryOrCreate | 给定的目录不存在则新建
Directory | 给定的路径必须是一个存在的目录
FileOrCreate | 给定的文件不存在则新建
File | 给定的路径必须是一个存在的文件
Socket | 给定的路径必须是一个存在的 UNXI socket
CharDevice | 给定的路径必须是一个存在的字符设备
BlockDevice | 给定的路径必须是一个存在的块设备


##### 3. 持久化存储
PersistentVolumes k8s持久化卷，旨在集群之外存储数据。


#### PV PersistentVolumes 
> k8s持久化存储的方案，pv是全局的资源，不和namespace绑定。

k8s持久卷主要是什么卷的访问方式，容量，回收策略和卷的具体的实现的一些参数。

##### pv的几种访问模式

模式 | 说明
---|---
ReadWriteOnce | 只能被单节点读写，可以被同一个节点的多个pod读写
ReadOnlyMany | 可被多个节点读
ReadWriteMany | 可被多个节点读写
ReadWriteOncePod | 只能被单个pod读写

##### pv的回收策略
策略 | 说明
--- | ---
Retain | PV 数据保留，但会一直处于 Released 的状态， 不能被其他PVC使用，除非手动释放PV。
Delete（默认） | pvc删除后，关联的pv也将被删除
Recycle（废弃） | 自动回收。执行（rm -rf /thevolume/*）

##### pv的几种状态
状态 | 说明
--- | ---
Available | 自由的资源，没有被绑定到pvc
Bound | pv被绑定到pvc
Released | pvc已经被释放了，但是pv还没有被释放
Failed | 自动回收失败

##### pv和pvc绑定的规则
通过PVC定义的 ==accessModes== 读写权限，和 ==storage== 的定义，PVC会自动找到符合这些配置的PV进行绑定。

**==一个PV被PVC绑定后，不能被别的PVC绑定==**。

#### 静态创建pv
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: create-pv-test
spec:
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  capacity: 
    storage: 1Gi
  hostPath: 
    path: /opt/logs/create-pv-test
```

#### 动态创建pv（StorageClass）

Kubernetes提供了一套可以自动创建PV的机制，即Dynamic Volume Provisioning（动态PV）。而手动创建并管理的PV叫做Static Volume Provisioning（静态PV）。

> StorageClass描述了一类存储的参数，k8s根据这类参数动态创建PersistentVolumes。就类似于java的类，k8s根据类创建具体的pv实例。

StorageClass对象会定义下面两部分内容:
1. PV的属性。比如，存储类型，Volume的大小等。
2. 创建这种PV需要用到的存储插件，即存储制备器。

有了这两个信息之后，Kubernetes就能够根据用户提交的PVC，找到一个对应的StorageClass，之后Kubernetes就会调用该StorageClass声明的存储插件，进而创建出需要的PV。

##### 创建 StorageClass
```
# sc是全局的资源，不需要namespace
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata: 
  name: inus-sc-test
allowVolumeExpansion: true #允许扩展卷，只能增加
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: "ext4"
volumeBindingMode: Immediate
reclaimPolicy: Retain
```

##### 创建 pvc
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: inus-volv-test
  namespace: 5g
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: inus-sc-test
  resources:
    requests:
      storage: 2Gi
```

##### 使用 StorageClass 创建 pod
```
apiVersion: v1
kind: Pod
metadata:
  name: volv-test
  namespace: 5g
  labels:
    app: mysql-app
spec:
  restartPolicy: Always
  containers:
  - name: volv-test
    image: mysql:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /var/lib/mysql
    ports:
    - containerPort: 3306
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: root
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: inus-volv-test
```

##### 创建 deployment
> deplyment部署的pod在pod删除后会主动拉起服务

```
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: 5g
  name: mysql-deployment
spec:
  selector: 
    matchLabels: 
      app: mysql-app  # 选择的都是namespace中的labels
  replicas: 1
  minReadySeconds: 5
  strategy: 
    type: Recreate
  template: # 描述pod, 创建pod的模板
    metadata:
      name: mysql-app
      namespace: 5g
      labels:
        app: mysql-app
    spec:
      containers:
      - name: mysql-app
        image: mysql:latest
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: volv
          mountPath: /var/lib/mysql
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: root
      volumes:
      - name: volv
        persistentVolumeClaim:
          claimName: inus-volv-test
```

##### 创建service
```
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: 5g
spec:
  selector: 
    app: mysql-app
  type: NodePort
  ports:
  - name: mysql-service-port
    protocol: TCP
    port: 3306
    targetPort: 3306
    nodePort: 33063
```

### nfs 的几种使用方式

#### 1. 挂载到宿主机节点作为本地路径使用
```
  volumes:
  - name: mysql-volv
    hostPath: 
      path: /opt/nfsdata/mysql  # nfs服务器上的路径
```

#### 2. pod 直接使用
```
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: root 
    volumeMounts:
    - name: mysql-volv
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-volv
    nfs: 
      server: 192.168.1.142 # nfs服务器地址
      path: /opt/nfsdata/mysql  # nfs服务器上的路径
```

#### 3. 创建 StorageClass 使用
```
# 创建 nfs provisioner
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: ns-root
      containers:
      - name: nfs-client-provisioner
        image: registry.cn-beijing.aliyuncs.com/mydlq/nfs-subdir-external-provisioner:v4.0.0
        volumeMounts:   # 挂载数据卷到容器指定目录
        - name: nfs-volv
          mountPath: /persistentvolumes # 不需要修改
        env:
        - name: PROVISIONER_NAME
          value: nfs-provisioner    # 此处供应者名字供storageclass调用
        - name: NFS_SERVER
          value: 192.168.1.142  # 填入NFS的地址
        - name: NFS_PATH
          value: /opt/nfsdata/share # 填入NFS挂载的目录
      volumes:
      - name: nfs-volv
        nfs:
          server: 192.168.1.142 # 填入NFS的地址
          path: /opt/nfsdata/share  # 填入NFS挂载的目录

# 创建 StorageClass
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata: 
  name: mysql-sc
allowVolumeExpansion: true #允许扩展卷，只能增加
provisioner:nfs-provisioner
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  fromBackup: ""
  fsType: "ext4"
volumeBindingMode: Immediate
reclaimPolicy: Retain

# 申明 pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-volv-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: mysql-sc
  resources:
    requests:
      storage: 2Gi
      
# 使用
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: root 
    volumeMounts:
    - name: mysql-volv
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-volv
    persistentVolumeClaim: 
      claimName: mysql-volv-pvc
```


