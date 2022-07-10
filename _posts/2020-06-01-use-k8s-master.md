---
title: "Kubernetes Cluster의 Master Node에 Pod Deploy 하기"
date: 2020-06-01 00:00:00 +0900
categories: [DevOps, K8S]
tags: [K8S, MasterNode]
render_with_liquid: false
---

## Preface

기본적으로 Kubernetes Master Node에는 Pod가 Deploy 되지 않는다. (보안상의 이유로)

만약, Master Node (Control Plane)에 Pod를 Deploy 하고자 한다면, 아래와 같이 실행 한다.

```console
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Command

상기는 명령은 Cluster 내의 모든 Node에서 아래 Taint를 제거 하는것이다.

> node-role.kubernetes.io/master

## 결과

Output은 다음과 같다.

```shell
node "test-01" untainted
taint "node-role.kubernetes.io/master:" not found
taint "node-role.kubernetes.io/master:" not found
```

이후 Scheduler는 Cluster내의 모든 Node에 Pod를 Deploy 한다.

## 원본 링크

[K8S Doc Link](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/){:target="_blank"}
