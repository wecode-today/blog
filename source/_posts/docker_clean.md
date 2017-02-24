title: docker镜像未清理导致磁盘空间满
date: 2016/12/15 10:53:00
categories:
  - docker
tags:
  - clean
  - docker
  - shell
---

# 简介
我们的某个服务突然有一天，所有的测试环境全部执行失败，返回500。结果到某台测试服务器上一看，磁盘空间满了！

最后发现是大量的历史版本Docker镜像未清理导致的：

```
noah-docker.host.huawei.com/noah-service/hunter         <none>              8470735ac8e2        4 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              d88765ddc8fe        5 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              4aee0b83558a        6 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              add02e6af8f5        6 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              97eb4f2da485        6 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              95b8f4d92383        6 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              a2b51b2e138a        7 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              3b250877a8e1        7 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              3aa940434a99        7 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              68b1ec83803c        7 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              951db5450246        8 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              73b1750b31b9        8 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              13e7b553b4b7        8 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              4ea2c3cdc7bc        8 weeks ago         1.121 GB
noah-docker.host.huawei.com/noah-service/hunter         <none>              a5bc7cef75d0        8 weeks ago         1.121 GB
```

在网络上查询了之后，可以使用`docker rmi $(docker images -q -f dangling=true)`命令来清理untagged镜像。执行之后清理出大概25G的空间。

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda2       36G  5.4G   29G  16% /
```

# 优化部署
由于测试版本的应用，版本号总是`lastest`，因此每部署一次就留下了一个untagged镜像。所以我们把清理脚本加入部署阶段，部署最新版本之前先执行一次清理。

```
echo "Clean old docker images"
docker rm $(docker ps -a -q)
docker rmi $(docker images -q -f dangling=true)
```
这样就能减少大量的untagged镜像，节省磁盘空间。
