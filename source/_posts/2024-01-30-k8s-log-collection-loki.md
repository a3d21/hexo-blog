---
title: K8s日志收集Loki
date: 2024-01-30 11:11:53
tags: tech
---

主流的K8s日志收集方案ELK需要部署es集群，很重，需要的机器资源比我们运行服务的机器还多。
调研发现轻量级日志方案[loki](https://grafana.com/oss/loki/)，正好满足我们需求。

<!-- more -->

## 1 Helm安装loki

```
kubectl create ns loki
helm repo add grafana https://grafana.github.io/helm-charts
# 8.5.13修复 chrome bom字符bug
helm upgrade --install loki --namespace=loki grafana/loki-stack  --set grafana.enabled=true --set grafana.image.tag=8.5.13 

```

## 2 部署Ingress暴露访问
```
kubectl -n loki apply -f ingress.yaml

# 查看密码; 使用 admin + 密码登录
kubectl get secret --namespace loki loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: loki-grafana-ingress
spec:
  rules:
  - host: loki-grafana.XXX.com
    http:
      paths:
      - backend:
          service:
            name: loki-grafana
            port: 
              name: service
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - loki-grafana.XXX.com
    secretName: loki-grafana-XXX-tls
```

## 3 挂载硬盘持久化日志
loki提供多种[持久化方案](https://grafana.com/docs/loki/latest/operations/storage/)，但最直接的是硬盘。我们使用挂载硬盘方式。

```
# 查看配置
kubectl -n loki get statefulsets.apps loki -o yaml

# 修改 `storage` volumeClaimTemplates后重新创建
```


```yaml
# demo statefulsets
volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "gp2"
      resources:
        requests:
          storage: 100Gi
```

## 4 修改日志保存时间
loki配置记录在loki secrets，可以改成configmap并修改配置
```
# 查看并提取日志
kubectl get secrets loki -n loki -o "jsonpath={.data['loki\.yaml']}" | base64 -d | tee loki.yaml
```

修改保存时间
```yaml
    table_manager:
      retention_deletes_enabled: true
      retention_period: 168h    //需要24h的整数倍
```

创建configmap
```
kubectl create cm -n loki loki-conf --from-file=loki.yaml=loki.yml
```

更新loki statefulsets，读configmap配置
```
kubectl -n loki edit statefulsets loki
```

等待服务更新完成，至此Loki部署完成。


## 相关资料
- [从ELK/EFK到PLG – 在EKS中实现基于Promtail + Loki + Grafana容器日志解决方案](https://aws.amazon.com/cn/blogs/china/from-elk-efk-to-plg-implement-in-eks-a-container-logging-solution-based-on-promtail-loki-grafana/)