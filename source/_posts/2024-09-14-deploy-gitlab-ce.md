---
title: Docker部署Gitlab CE
date: 2024-09-14 16:16:52
tags: tech
---

Gitlab是开发中一个重要的工具，本文整理Gitlab部署过程，方便未来查阅。

## 0 初始化机器
推荐RAM 8GB以上。

## 1 增加至少4GB swap
```sh
# 创建swap
dd if=/dev/zero of=/swapfile bs=4M count=1024
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# 查看swap状态
swapon -s 

# 自动挂载
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

## 2 挂载数据盘，建议nfsv4
```sh
sudo mount -t nfs -o vers=4.0,noresvport 10.0.0.1:/ /gitlabdata
echo '10.0.0.1:/  /gitlabdata  nfs  vers=4.0,noresvport  0  0' >> /etc/fstab
```

## 3 修改ssh端口号，防止端口冲突
```
vi /etc/ssh/sshd_config
# 修改 Port 为 2222，22端口保留给gitlab

# 重启机器
reboot 
```

## 4 docker跑gitlab-ce
```sh
GITLAB_HOME="/gitlabdata"
docker run --detach --hostname git.xxx.com  \
    --env GITLAB_OMNIBUS_CONFIG="external_url 'http://git.xxx.com'" \       # 自定义域名，指向本机器
    --env TZ="Asia/Shanghai"   -v /etc/localtime:/etc/localtime  \
    --publish 443:443 --publish 80:80 --publish 22:22 --publish 5050:5050 \
    --name gitlab   --restart always   --volume $GITLAB_HOME/config:/etc/gitlab:Z   \
    --volume $GITLAB_HOME/logs:/var/log/gitlab:Z   --volume $GITLAB_HOME/data:/var/opt/gitlab:Z   \
    --shm-size 1024m   gitlab/gitlab-ce:17.2.1-ce.0

# 初始化完成后
# 查看root密码
docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
```



## 5 开启ssl
```
# 使用let's encrypt签证书
vi /gitlabdata/gitlab/config/gitlab.rb

#修改 external_url 为 htts://xxx.xxx.com 自定义域名
#letsencrypt['enable'] = true
#letsencrypt['auto_renew'] = true
#letsencrypt['auto_renew_hour'] = 120

应用配置
docker exec -t gitlab gitlab-ctl reconfigure
```

## 5 配置container registry
配置CR域名指向gitlab机器，修改配置。
```
registry_external_url 'https://gcr.xxx.xxx'
gitlab_rails[registry_enable'] = true
gitlab_rails[registry_host'] = 'gcr.xxx.xxx'
gitlab_rails[registry_port'] = 443
```
应用配置
```
docker exec -t gitlab gitlab-ctl reconfigure
```

## 完结
至此，Gitlab部署完成。