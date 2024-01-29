---
title: K8s Gitlab CI/CD
date: 2024-01-29 18:25:26
tags: tech
---

记录&分享K8s Gitlab CI/CD搭建过程。

<!-- more -->

## 1 helm安装gitlab runner
```sh
helm repo add gitlab https://charts.gitlab.io
helm repo update gitlab
kubectl create ns gitlab
helm install -n gitlab -f values.yaml gitlab-runner gitlab/gitlab-runner
```

values.yaml参考
``` yaml
# values.yaml
runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        # Run all containers with the privileged flag enabled.
        # See https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnerskubernetes-section for details.
        image = "goland:1.21"
        service_account = "gitlab-runner"   # 指定sa运行job
        privileged = true
rbac:
  create: true

gitlabUrl: URL
runnerToken: TOKEN
```

## 2 授权gitlab-runner用户修改deployment权限，用于更新服务

我们gitlab-runner跑在测试集群上，直接授权ClusterRole。
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dp-role
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: dp-role-binding

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dp-role
subjects:
  - kind: ServiceAccount
    name: "gitlab-runner"
    namespace: gitlab
```

## 3 在job里直接用`bitnami/kubectl`镜像更新服务
```yaml
# .gitlab-ci.yaml
upgrade-service:
  stage: deploy
  image:
    name: bitnami/kubectl:1.28
    entrypoint: [ "" ]
  script:
    - kubectl -n default set image deployment XXX XXX=IMAGE:VERSION
```

## 4 跨集群更新服务

使用vault或env保存kubectl配置，使用`bitnami/kubectl`调用。
``` yaml
# .gitlab-ci.yaml
upgrade-service:
  stage: deploy
  image:
    name: bitnami/kubectl:1.28
    entrypoint: [ "" ]
  script:
    - echo $K8S_CONF|base64 -d > /.kube/config # env保存配置
    - kubectl -n default set image deployment XXX XXX=IMAGE:VERSION
```