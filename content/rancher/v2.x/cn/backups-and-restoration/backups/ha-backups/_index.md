---
title: 集群备份
weight: 50
---

本节介绍在Rancher HA下如何备份数据。

## **先决条件:**

- Rancher Kubernetes Engine v0.1.7或更高版本，

    RKE v0.1.7以及更高版本才支持`etcd`快照功能。

- rancher-cluster.yml

    需要使用安装Rancher的RKE配置文件rancher-cluster.yml，将此文件放在与RKE二进制文件同级目录中。

- 你必须将每个`etcd`节点还原到同一快照。在运行`etcd snapshot-restore`命令之前，将你正在使用的快照从一个节点复制到其他节点

## 1. 创建`etcd`数据快照

有两种方案创建`etcd`快照: 定时自动创建快照和或手动创建快照，每种方式对应特定的场景。

- 方案 A: 定时自动创建快照

    在Rancher HA安装后，我们建议配置RKE以定时(默认5分钟)自动创建快照，以便始终拥有可用的安全恢复点。

- 方案 B: 手动创建快照

    我们建议在升级或恢复其他快照等事件之前创建一次性快照。

### 方案 A: 定时自动创建快照

对于通过RKE高可用安装的Rancher，我们建议开启定时自动创建快照，以便始终拥有安全的恢复点。

定时自动创建快照服务是RKE附带的服务，默认没有开启。可以通过在rancher-cluster.yml中添加配置来启用etcd-snapshot(定时自动创建快照)服务。

**启用定时自动创建快照:**

1. 编辑`rancher-cluster.yml`配置文件；

2. 在`rancher-cluster.yml`配置文件中添加以下代码：

    ```yaml
    services:
      etcd:
        snapshot: true  # 是否启用快照功能，默认false；
        creation: 6h0s  # 快照创建间隔时间，不加此参数，默认5分钟；
        retention: 24h  # 快照有效期，此时间后快照将被删除；
    ```

3. 根据实际需求跳转以上代码参数；

4. 保存并关闭`rancher-cluster.yml`；

5. 打开**Terminal**并切换路径到RKE二进制文件所在目录.确保`rancher-cluster.yml`也在这个路径下；

6. 运行以下命令：

    ```bash
    # MacOS
    ./rke_darwin-amd64 up --config rancher-cluster.yml
    # Linux
    ./rke_linux-amd64 up --config rancher-cluster.yml
    ```

### 方案 B: 手动创建快照

当你即将升级Rancher或将其恢复到以前的快照时，你应该对数据手动创建快照，以便数据异常时可供恢复。

**手动创建快照:**

1. 打开**Terminal**并切换路径到RKE二进制文件所在目录.确保`rancher-cluster.yml`也在这个路径下；

2. 输入以下命令：

    ```bash
    # MacOS
    ./rke_darwin-amd64 etcd snapshot-save --name <SNAPSHOT.db> --config rancher-cluster.yml
    # Linux
    ./rke_linux-amd64 etcd snapshot-save --name <SNAPSHOT.db> --config rancher-cluster.yml
    ```
    >注意：替换<SNAPSHOT.db>为你喜欢的名称，例如：<SNAPSHOT.db>

3. RKE会获取每个`etcd`节点的快照，并保存在`/opt/rke/etcd-snapshots`目录下；

## 2. 备份快照到安全位置

在创建快照后，应该把它保存到安全的地方，以便在群集遇到灾难情况时快照不受影响，这个位置应该是持久的。
