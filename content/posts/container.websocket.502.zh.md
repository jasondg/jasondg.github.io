---
title: "端口映射后容器内 WebSocket 产生 502 Bad Gateway 错误"
date: 2022-03-30T19:59:15+08:00
categories: ["error"]
tags: ["container", "websocket", "proxy", "Podman", "Nginx"]
draft: false
---

# 前言

前段时间试着把一些老服务升级了一下，有一些也移进了容器里，用 Podman 跑 rootless container。因为外层的 Nginx 没有动，本以为能快速迁移，但启动 container 后服务并未正常运行，执行`systemctl --user status someService`显示服务为`active`，并没有错误信息。重新尝试后仍然存在问题,客户端出现 `502 Bad Gateway` 错误，遂进行问题定位。

# 定位问题

## 相关配置

因为仅是部分服务迁移到容器，原本的 Nginx 服务并没有变动，相关配置为

```
location /somePath {
    proxy_redirect off;
    proxy_pass http://127.0.0.1:8000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $http_host;
}
```

容器使用 Podman 在普通用户下运行，执行参数为

```
-p 8000:3000
-v ./someDir/:/etc/some/config/:rw
```

服务本声也并没有变动，相关配置与数据也维持原样并进行映射。

## 错误信息

客户端查看 log 后发现报错：

```
Failed to dial to wss://someSite/somePath
502 Bad Gateway
websocket: bad handshake
```

但没有额外信息，初步判断为 Nginx 无法连通子服务，从而产生 502 Bad Gateway 错误。

## 问题排查

- 测试确认 Nginx 能够正常运行，其他路径下非容器内的服务都正常运行
- 在同参数下运行`podman run busybox ls -al`，确认配置文件与数据能被 container 正常获取
- host 下直接连接 container 的 WebSocket，失败，显示 502
- `systemctl --user status`显示 container 正常运行

于是缩小了问题范围，应该是容器化后的网络映射导致了问题。

# 解决问题

查阅 podman 文档[^podman],在 [podman run](https://docs.podman.io/en/latest/markdown/podman-run.1.html) 的文档中找到 [`--publish`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#publish-p-ip-hostport-containerport-ip-containerport-hostport-containerport-containerport) 与[`--network`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#network-mode-net)的说明。其中列出了网络模式：

> Valid mode values are:
>
> - _bridge[:OPTIONS,…]_: Create a network stack on the default bridge. This is the default for rootfull containers. It is possible to specify these additional options:
>   For example to set a static ipv4 address and a static mac address, use --network bridge:ip=10.88.0.10,mac=44:33:22:11:00:99.
> - _<network name or ID>[:OPTIONS,…]_: Connect to a user-defined network; this is the network name or ID from a network created by podman network create. Using the network name implies the bridge network mode. It is possible to specify the same options described under the bridge mode above. You can use the --network option multiple times to specify additional networks.
> - _none_: Create a network namespace for the container but do not configure network interfaces for it, thus the container has no network connectivity.
> - _container:id_: Reuse another container’s network stack.
> - _host_: Do not create a network namespace, the container will use the host’s network. Note: The host mode gives the container full access to local system services such as D-bus and is therefore considered insecure.
> - _ns:path:_ Path to a network namespace to join.
> - _private_: Create a new namespace for the container. This will use the bridge mode for rootfull containers and slirp4netns for rootless ones.
> - _slirp4netns[:OPTIONS,…]_: use slirp4netns(1) to create a user network stack. This is the default for rootless containers. It is possible to specify these additional options:

并且检索后找到 stackoverflow 上的一个回答[^sof]，指出容器化会创建 network namespace，所以容器内部获取的 localhost 并非 host 下的 localhost，从而导致容器内服务不能正常运行。

通过设置`--net=host`运行容器，或修改 Nginx 中`proxy_pass`的配置指向实际的 host 网络，就可以解决 502 的问题。

# 一些延伸

使用`--net=host`参数是最简便的解决方法，但直接使用 host 网络会导致 port 冲突，造成多 container 场景下的部署问题。对容器网络进行配置的方式应该是更好的，但对于简单场景来说有些不必要，以后有机会也可以尝试一下。

单机使用容器也是抱着尝试的心态，看看是否能简化服务的管理。使用过程中发现对容器相关的 cgroup、网络等问题还不够熟悉，就先记录下来，后续有时间时可以学习一下相关知识。

[^podman]: https://docs.podman.io/en/latest/
[^sof]: https://stackoverflow.com/questions/38346847/nginx-docker-container-502-bad-gateway-response
