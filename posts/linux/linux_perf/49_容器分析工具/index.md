# 5.8 容器的性能分析工具

本节我们来介绍一些专门用于容器性能分析的工具。
<!-- more -->

## 1. 工具简介
本节我们将介绍如下一些性能分析工具的使用:
1. nsenter: 可以进入容器命名空间
2. sysdig: 用于容器的动态追踪，汇集了一些列性能工具的优势

## 2. nsenter
nsenter
- 作用: 可以进入容器命名空间

```bash
# 1. 安装
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter

# 2. 使用
# 由于这两个容器共享同一个网络命名空间，所以我们只需要进入app的网络命名空间即可
$ PID=$(docker inspect --format {{.State.Pid}} app)
# -i表示显示网络套接字信息
$ nsenter --target $PID --net -- lsof -i
COMMAND    PID            USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
redis-ser 9085 systemd-network    6u  IPv4 15447972      0t0  TCP localhost:6379 (LISTEN)
redis-ser 9085 systemd-network    8u  IPv4 15448709      0t0  TCP localhost:6379->localhost:32996 (ESTABLISHED)
python    9181            root    3u  IPv4 15448677      0t0  TCP *:http (LISTEN)
python    9181            root    5u  IPv4 15449632      0t0  TCP localhost:32996->localhost:6379 (ESTABLISHED)
```

## 3. sysdig

