---
title: kubernetes 网络隔离
categories: kubernetes
tags: kubernetes NetworkPolicy
---

> 设置 POD 的网络访问策略，即 POD 的入网和出网策略。
> NetworkPolicy 是 K8s 提供 POD 网络隔离能力的资源。（类似于设置网络防火墙规则）

POD 默认的允许所有的入网和出网流量的。如果要限制这一规则，则需要 NetworkPolicy 来指定规则。

#### 示例分析
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.36.1
  name: prometheus-k8s
  namespace: monitoring
spec:
  egress:
  - {}
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - port: 9090
      protocol: TCP
    - port: 8080
      protocol: TCP
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: grafana
    ports:
    - port: 9090
      protocol: TCP
  podSelector:
    matchLabels:
      app.kubernetes.io/component: prometheus
      app.kubernetes.io/instance: k8s
      app.kubernetes.io/name: prometheus
      app.kubernetes.io/part-of: kube-prometheus
  policyTypes:
  - Egress
  - Ingress
```
spec.podSelector 指定该 NetworkPolicy 资源涉及的 POD 的范围。在 NetworkPolicy 命名空间中的对应的labels的 POD 将会执行上诉配置。\
spec.podSelector: {} 将会选择所有的 POD。

ingress 配置 POD 的数据流入的规则。即入网白名单。
from 和 port 配置指定 POD 的入网端口。
```
spec:
  ingress:
    # 配置入网的pod
  - from:
    # 配置入网ip地址段
    - ipBlock:
        cidr: 172.17.0.0/16
        # 排除指定的ip地址段
        except:
        - 172.17.1.0/24
    # 配置命名空间下所有的 POD
    - namespaceSelector:
        matchLabels:
          namespaceLables: namespaceValue
    # 配置命名空间下指定的 POD
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    # 指定pod的入网的端口号
    ports:
    - port: 9090
      protocol: TCP
    - port: 8080
      protocol: TCP
```

egress 配置数据的出网规则
```
spec:
  # 配置哪些pod需要遵循该规则
  egress:
  - to:
    # 配置pod的ip地址段
    - ipBlock:
        cidr: 10.0.0.0/24
    # 配置pod的出网端口号
    ports:
    - protocol: TCP
      port: 5978
```

policyTypes 配置 NetworkPloicy 规则生效的策略。入网、出网还是入网和出网。

#### NetworkPloicy 支持的网络插件
要让网络策略生效，就需要特定的网络插件支持，目前已经实现了 NetworkPolicy 的网络插件包括 Calico、Weave 和 kube-router 等项目，但是并不包括 Flannel 项目。

如果需要 Flannel 支持 NetworkPloicy 需要安装额外的 [插件](https://projectcalico.docs.tigera.io/getting-started/kubernetes/flannel/flannel)




