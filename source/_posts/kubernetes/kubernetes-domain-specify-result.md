---
title: 容器内指定特定域名解析结果的几种方式
date: 2023-07-10 16:37:33
tags:
  - Kubernetes
---

在本篇文章中，我们将探讨如何在容器内指定特定域名解析结果的几种方式。为了方便演示，首先我们创建一个演示用的Deployment配置文件。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-deployment
  labels:
    app: busybox
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        args:
        - /bin/sh
        - -c
        - "while true; do echo Hello, Kubernetes!; sleep 10;done"
```

这个deployment会创建1个busybox的pod，容器每隔10s会打印“Hello, Kubernetes!”到控制台

## TL;DR

| 方案              | 修改级别                   | 是否推荐  | 备注                 |
|-----------------|------------------------|-------|--------------------|
| 修改/etc/hosts    | pod                    | 否     |                    |
| 添加HostAliases记录 | pod/deploy/statefulset | 是     |                    |
| 修改Coredns配置     | 整个kubernetes集群         | 是     |                    |
| 自定义DNS策略        | pod/deploy/statefulset | 视情况而定 | 如需对接三方的DNS服务器，推荐采用 |
| 使用三方DNS插件       | 整个kubernetes集群         | 否     | 不推荐，Coredns为业内主流   |

## 修改/etc/hosts

修改/etc/hosts是最传统的方式，直接在容器内修改相应的文件来实现域名解析，在Pod级别生效。由于其可维护性较差（每次pod发生重启都需要手动修改），不推荐在生产环境使用。

例如，我们可以在/etc/hosts里面添加这样一条记录

```
250.250.250.250 four-250
```

```
/ # ping four-250
PING four-250 (250.250.250.250): 56 data bytes
```

## 添加HostAliases记录

HostAliases是kubernetes中Pod配置的一个字段，它提供了Pod内容器的`/etc/hosts`文件的附加记录。这在某些情况下非常有用，特别是当你想要覆盖某个主机名的解析结果，或者提供网络中没有的主机名解析时。

这个可以在Pod、Replica、Deployment、StatefulSet的级别修改，维护性稍强。举个🌰，我们将上面的yaml修改为

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox-deployment
  labels:
    app: busybox
spec:
  replicas: 3
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      hostAliases:
      - ip: "250.250.250.250"
        hostnames:
        - "four-250"
      containers:
      - name: busybox
        image: busybox
        args:
        - /bin/sh
        - -c
        - "while true; do echo Hello, Kubernetes!; sleep 10;done"
```

这个时候我们查看容器的/etc/hosts，发现它被kubernetes自动插入了一条记录**Entries add by HostAliases。**这就是hostAliases的实现原理

在**kubelet_pods**代码中进行了这样的写入动作

```go
func hostsEntriesFromHostAliases(hostAliases []v1.HostAlias) []byte {
	if len(hostAliases) == 0 {
		return []byte{}
	}

	var buffer bytes.Buffer
	buffer.WriteString("\n")
	buffer.WriteString("# Entries added by HostAliases.\n")
	// for each IP, write all aliases onto single line in hosts file
	for _, hostAlias := range hostAliases {
		buffer.WriteString(fmt.Sprintf("%s\t%s\n", hostAlias.IP, strings.Join(hostAlias.Hostnames, "\t")))
	}
	return buffer.Bytes()
}
```

## Coredns配置

我们可以通过修改ConfigMap来实现让容器解析特定域名的目的。

### 更改Coredns配置

我们可以通过以下命令修改Coredns的配置：

```bash
kubectl edit cm coredns -n kube-system
```

### 原有的configmap

```yaml
Corefile: |
    .:53 {
        log
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        hosts {
           192.168.65.2 host.minikube.internal
           fallthrough
        }
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

在hosts里面加上特定的记录

```
250.250.250.250 four-250
```

如果您没有配置reload插件，则需要重启Coredns才能生效，默认的reload时间是30s，在plugin/reload/setup.go的defaultInterval中定义

## 自定义DNS策略

通过修改DNS策略。使得对于单个Pod/Deploy/StatefulSet将特定的域名解析发给特定的服务器来达到效果，如下，可以对pod添加dns的服务器以及search域

```yaml
   spec:
      dnsConfig:
        nameservers:
          - 1.2.3.4
        searches:
          - search.prefix
      containers:
      - name: busybox
        image: busybox
        args:
        - /bin/sh
        - -c
        - "while true; do echo Hello, Kubernetes!; sleep 10;done"
```

## 使用第三方DNS插件

不推荐，使用其他的DNS插件，来做一些炫酷的自定义操作。而且目前Coredns也是业内的主流，没有很好的替代
