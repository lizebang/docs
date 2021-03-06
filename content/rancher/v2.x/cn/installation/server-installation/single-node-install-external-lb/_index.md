---
title: 单节点安装+外部负载平衡器
weight: 2
---

对于开发环境，我们推荐通过运行一个Docker容器来安装Rancher。在此场景中，你将使用单个Docker容器将Rancher部署到Linux主机。然后，你将配置外部负载均衡器以与Rancher配合使用。

> 查看 [单节点安装]({{< baseurl >}}/rancher/v2.x/cn/installation/server-installation/single-node-install).

##  一、配置Linux主机

配置一台Linux主机来运行Rancher server。

### 要求

1. #### 操作系统

    - Ubuntu 16.04（64位）
    - 红帽企业Linux 7.5（64位）
    - RancherOS 1.3.0（64位）

2. #### 硬件

    硬件需求根据Rancher部署的规模进行扩展。根据需求配置每个节点。

    | 部署大小 | 集群(个)  | 节点(个) | vCPU                                            | 内存 |
    | -------- | --------- | -------- |     ------------------------------------------- | ---- |
    | 小       | 不超过10  | 最多50   | 2C                                              | 4GB  |
    | 中       | 不超过100 | 最多500  | 8C                                              | 32GB |
    | 大       | 超过100   | 超过500  | [联系Rancher](https://rancher.com/contact/) |      |

3. #### 软件

    - Docker

      > **注意：**如果你使用的是RancherOS，请确保你将Docker引擎切换为受支持的版本`sudo ros engine switch docker-17.03.2-ce`

      **支持的Docker版本**

      - `1.12.6`
      - `1.13.1`
      - `17.03.2`

      [Docker安装说明](https://docs.docker.com/install/)

      > **注意：** 该`rancher/rancher`镜像托管在[DockerHub上](https://hub.docker.com/r/rancher/rancher/tags/)。如果你无法访问DockerHub，或者离线环境下安装Rancher，请参阅[Air Gap安装](/docs/rancher/v2.x/cn/installation/server-installation/air-gap-installation/)。
      >
      > 有关可用的其他Rancher server标记的列表，请参阅[Rancher server tags](/docs/rancher/v2.x/cn/installation/server-tags/)。

4. #### 端口

    下图描述了Rancher的基本端口要求。有关全面列表，请参阅[端口要求](/docs/rancher/v2.x/cn/installation/references/)。

      ![基本端口要求](/docs/img/rancher/port-communications.png)

## 二、安装Rancher并配置SSL证书

出于安全考虑，使用Rancher时需要SSL镜像加密。SSL可以保护所有Rancher网络通信，例如登录或与集群交互。

> **注意Air Gap用户：**如果你正在访问此页面以完成[Air Gap安装](/docs/rancher/v2.x/cn/installation/server-installation/air-gap-installation/)，在运行安装命令时，必须在Rancher镜像前面加上你私有仓库的地址，替换`<REGISTRY.DOMAIN.COM:PORT>`为你的私有仓库地址。
>
>例如: <REGISTRY.DOMAIN.COM:PORT>/rancher/rancher:latest

1. ### 方案A-使用你自己的自签名证书

    如果你选择使用自签名证书来加密通信，则必须将证书安装在负载均衡器上，并且将CA证书放置于Rancher 容器中。运行docker命令来部署Rancher，并将其指向你的证书。

    > **先决条件:**
    > - 创建一个自签名证书;
    > - 证书文件必须是 **PEM** 格式;

    **安装Rancher:**

    Rancher安装可以使用你提供的自签名证书来加密通信。在运行Docker命令来部署Rancher时，将Docker指向你的CA证书文件。

    ```bash
    docker run -d --restart=unless-stopped \
    -p 80:80 -p 443:443 \
    -v /etc/your_certificate_directory/cacerts.pem:/etc/rancher/ssl/cacerts.pem \
    rancher/rancher:latest
    ```

2. ### 方案B-使用权威CA机构颁发的证书

    如果你公开发布你的应用，理想情况下应该使用由权威CA机构颁发的证书。

    > **先决条件：**
    >
    > - 证书文件必须是[PEM格式]    (/docs/rancher/v2.x/cn/installation/server-installation/single-node-install-external-lb/#我如何知道我的证书是否为pem格式)。

    **安装Rancher:**

    如果你使用由权威CA机构颁发的证书，则无需在Rancher容器中安装你的CA证书。只需运行下面的基本安装命令即可。

    ```bash
    docker run -d --restart=unless-stopped \
    -p 80:80 -p 443:443 \
    rancher/rancher:latest
    ```

## 三、配置七层负载均衡器

在Rancher容器前使用负载平衡器时，不需要容器将端口通信从端口80重定向到端口443。通过 `X-Forwarded-Proto: https`重定向，端口重定向被禁用。

负载均衡器或代理必须配置为支持以下内容：

- **WebSocket** 连接

- **SPDY**/**HTTP/2** 协议

- 传递/设置以下headers:

    | Header | Value | Description |
    |--------|-------|-------------|
    | `Host`                | Hostname used to reach Rancher.          | To identify the server requested by the client.
    | `X-Forwarded-Proto`   | `https`                                  | To identify the protocol that a client used to connect to the load balancer or proxy.<br /><br/>**Note:** If this header is present, `rancher/rancher` does not redirect HTTP to HTTPS.
    | `X-Forwarded-Port`    | Port used to reach Rancher.              | To identify the protocol that client used to connect to the load balancer or proxy.
    | `X-Forwarded-For`     | IP of the client connection.             | To identify the originating IP address of a client.

### 四、Nginx 配置文件示例

此Nginx配置文件在Nginx version 1.13 (mainline) and 1.14 (stable)通过测试

```bash
upstream rancher {
    server rancher-server:80;
}

map $http_upgrade $connection_upgrade {
    default Upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name rancher.yourdomain.com;
    ssl_certificate /etc/your_certificate_directory/fullchain.pem;
    ssl_certificate_key /etc/your_certificate_directory/privkey.pem;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://rancher;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # This allows the ability for the execute shell window to remain open for up to 15 minutes. Without this parameter, the default is 1 minute and will automatically close.
        proxy_read_timeout 900s;
    }
}

server {
    listen 80;
    server_name rancher.yourdomain.com;
    return 301 https://$server_name$request_uri;
}
```

## 五、删除默认CA证书

>注意: 此操作仅适用于使用权威CA机构颁发的证书，如果你使用的是自签名证书，请不要进行此过程。

默认情况下，Rancher会在安装后自动为自己生成自签名CA证书。但是，由于你提供了自己的证书，因此必须禁用Rancher自动生成的CA证书。

**删除默认证书:**

1. 登录Rancher。
2. 选择 **设置** > **cacerts**。
3. 选择`Edit`并删除内容。然后点击`Save`。

## 六、下一步？

你有几个选择：

- 创建Rancher server的备份：[单节点备份和恢复](/docs/rancher/v2.x/cn/backups-and-restoration/single-node-backup-and-restoration/)。
- 创建一个Kubernetes集群：[创建一个集群](/docs/rancher/v2.x/cn/installation/server-installation/single-node-install/%7B%7B%20%3Cbaseurl%3E%20%7D%7D/rancher/v2.x/cn/tasks/clusters/creating-a-cluster/)。

## 七、FAQ and Troubleshooting

1. ### 如何知道我的证书是否为PEM格式？

    你可以通过以下特征识别PEM格式：

      - 该文件以下列标题开头：
      `-----BEGIN CERTIFICATE-----`
      - 标题后面跟着一串长字符
      - 该文件以页脚结尾：
      `-----END CERTIFICATE-----`

    **PEM证书示例:**

      ```bash
      ----BEGIN CERTIFICATE-----
      MIIGVDCCBDygAwIBAgIJAMiIrEm29kRLMA0GCSqGSIb3DQEBCwUAMHkxCzAJBgNV
      ... more lines
      VWQqljhfacYPgp8KJUJENQ9h5hZ2nSCrI+W00Jcw4QcEdCI8HL5wmg==
      -----END CERTIFICATE-----
      ```

2. ### 如果我想添加我的中间证书，证书的顺序是什么？

    添加证书的顺序如下：

      ```bash
      -----BEGIN CERTIFICATE-----
      %YOUR_CERTIFICATE%
      -----END CERTIFICATE-----
      -----BEGIN CERTIFICATE-----
      %YOUR_INTERMEDIATE_CERTIFICATE%
      -----END CERTIFICATE-----
      ```

3. ### 我如何验证我的证书链？

    你可以使用`openssl`二进制验证证书链。如果该命令的输出（参见下面的命令示例）结束`Verify return code: 0 (ok)`，那么证书链是有效的。该`ca.pem`文件必须与你添加到`rancher/rancher`容器中的文件相同。当使用由认可的认证机构签署的证书时，可以省略该`-CAfile`参数。

    **命令:**

      ```bash
      openssl s_client -CAfile ca.pem -connect rancher.yourdomain.com:443
      ...
      Verify return code: 0 (ok)
      ```

4. ### 数据持久

    Rancher `etcd`用作数据存储，使用单节点安装时，将使用内置`etcd`。数据持久位于容器中的以下路径中： `/var/lib/rancher`。你可以将主机卷挂载到此位置以保留其运行的数据。

    **命令**:

      ```bash
      # 指定主机路径
      HOST_PATH=xxxx
      docker run -d --restart=unless-stopped \
      -p 80:80 -p 443:443 \
      -v $HOST_PATH:/var/lib/rancher \
      rancher/rancher:latest
      ```