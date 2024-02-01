---
title: K8s Traefik Ingress
date: 2024-02-01 10:28:09
tags: [tech, k8s]
---

Traefik是K8s Ingress的一种方案，支持丰富的中间件扩展方案。

<!-- more -->

## 1 helm 安装traefik

```sh
kubectl create ns traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
# 生产环境建议使用DaemonSet部署
helm install -n traefik traefik traefik/traefik --set deployment.kind=DaemonSet 
```

## 2 配置LB
查看**EXTERNAL-IP**，并配置domain解析到LB地址。
```sh
kubectl get service -n traefik
```

## 3 查看dashboard状态

```sh
# 访问 http://localhost:9000/dashboard/
kubectl -n traefik port-forward traefik-XXXX 9000:9000
```

