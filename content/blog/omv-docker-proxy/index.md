---
title: OMV配置docker网络代理
date: 2025-08-11 20:00:00+0800
tags:
    - OMV
    - OpenMediaVault
    - docker
---

解决国内docker无法正常登录和拉取镜像的问题。

{{< callout type="info" >}}
  此处直接给docker挂个代理(比如clash)，适用于不想更改docker仓库的情况。
{{< /callout >}}

1. 编辑文件
    ```yaml {filename="/etc/systemd/system/docker.service.d/http-proxy.conf"}
    [Service]
    Environment="HTTP_PROXY=http://proxy-ip:port/"
    Environment="HTTPS_PROXY=http://proxy-ip:port/"
    ```
2. 执行`systemctl daemon-reload`
3. 执行`systemctl restart docker`

此时，docker应该可以正常走代理了
