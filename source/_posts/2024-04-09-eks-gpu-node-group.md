---
title: EKS部署显卡集群
date: 2024-04-09 18:49:34
tags: [tech, k8s]
---

## 背景
部署共享GPU集群，加速AI应用计算。

## 步骤
### 1 创建还GPU的node-group
AMI选择AL2_x86_64_GPU，自带显卡需要，省去手动安装步骤。
建议选择nvidia显卡机器。
节点打上tag: `eks-node=gpu`，后面安装驱动用。

### 2 helm安装 nvidia-device-plugin

```shell

helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

helm upgrade -i nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  -f nvdia-plugin-values.yaml \
  --create-namespace \
  --version 0.14.5
```


nvdia-plugin-values.yaml示例，选择带显卡机器。
```
nodeSelector: 
  eks-node: gpu
```

### 3 验证安装
```shell
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
    - name: cuda-container
      image: nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda10.2
      resources:
        limits:
          nvidia.com/gpu: 1 # requesting 1 GPU
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
EOF

$ k logs gpu-pod

```

## 参考资料
- https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/
- https://github.com/NVIDIA/k8s-device-plugin#quick-start
- https://aws.amazon.com/blogs/containers/gpu-sharing-on-amazon-eks-with-nvidia-time-slicing-and-accelerated-ec2-instances/