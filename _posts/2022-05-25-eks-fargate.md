---
title: "EKS Fargate Cluster with Ingress ALB"
date: 2022-05-25 00:00:00 +0900
categories: [DevOps, K8S]
tags: [K8S, eksctl, ingress, Kubernetes]
render_with_liquid: false
---

## 설치

```console
$ brew install awscli
$ brew install eksctl
$ brew install helm
```

## AWS CLI Login

```console
$ aws configure
AWS Access Key ID [None]: xxx
AWS Secret Access Key [None]: xxx
Default region name [None]: ap-northeast-2
Default output format [None]: json
```

## EKS Cluster 생성

- cluster-fargate.yaml 파일 생성

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: fargate-cluster
  region: ap-northeast-2

fargateProfiles:
  - name: fc-default
    selectors:
      # All workloads in the "default" Kubernetes namespace will be
      # scheduled onto Fargate:
      - namespace: default
      # All workloads in the "kube-system" Kubernetes namespace will be
      # scheduled onto Fargate:
      - namespace: kube-system
  - name: fc-dev
    selectors:
      # All workloads in the "dev" Kubernetes namespace matching the following
      # label selectors will be scheduled onto Fargate:
      - namespace: dev
        labels:
          type: dev
```

- ecksctl 명령어로 Cluster 생성

```console
$ eksctl create cluster -f cluster-fargate.yaml
```

## IAM Policy Setting 및 K8S ServiceAccount 생성

[AWS 공식 가이드](https://aws.amazon.com/ko/premiumsupport/knowledge-center/eks-alb-ingress-controller-fargate/)

- 클러스터가 서비스 계정에 IAM를 사용하도록 허용

```console
$ eksctl utils associate-iam-oidc-provider --cluster YOUR_CLUSTER_NAME --approve
```

- AWS Load Balancer Controller에서 사용자 대신 AWS API를 호출하는 것을 허용하는 IAM 정책을 다운로드하고 이를 적용

  - IAM Policy Download
```console
$ curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json
```

  - IAM Policy Creation
```console
$ aws iam create-policy \
   --policy-name AWSLoadBalancerControllerIAMPolicy \
   --policy-document file://iam_policy.json
```

  - kube-system 네임스페이스에 aws-load-balancer-controller라는 이름의 ServiceAccount 를 생성, 이때 위에서 생성한 Policy도 지정
```console
$ eksctl create iamserviceaccount \
  --cluster=YOUR_CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

  - 확인
```console
$ eksctl get iamserviceaccount --cluster YOUR_CLUSTER_NAME --name aws-load-balancer-controller --namespace kube-system
or
$ kubectl get serviceaccount aws-load-balancer-controller --namespace kube-system
```

## AWS Load Balancer Controller 설치

- Helm Repository 추가

```console
$ helm repo add eks https://aws.github.io/eks-charts
```

- TargetGroupBinding CRD 설치

```console
$ kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

- Helm Chart로 LoadBalancer 설치

```console
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=YOUR_CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set region=YOUR_REGION_CODE \
    --set vpcId=<VPC_ID> \
    --set serviceAccount.name=aws-load-balancer-controller \
    -n kube-system
```

## Test Application 배포로 확인해보기

- fargate profile 생성

```console
$ eksctl create fargateprofile \
--cluster your-cluster \
--region your-region-code \
--name your-alb-sample-app \
--namespace game-2048
```

- 다음 파일을 저장하고, kubectl apply -f 해주자.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
        - image: alexwhen/docker-2048
          imagePullPolicy: Always
          name: app-2048
          resources:
            limits:
              cpu: 1
              memory: 1024Mi
            requests:
              cpu: 1
              memory: 1024Mi
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: ClusterIP
  selector:
    app.kubernetes.io/name: app-2048
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

- AWS Console 에서 ALB 가 생성됨을 확인하고, DNS 80포트로 잘뜨는지 확인해보자.

## R53 -> ALB Ingress, Https 정의법

- ACM으로 인증서를 생성해주자.
- R53의 DNS A Record에 ALB를 바라보게 세팅해 주고, 아래와 같이 생성한다.
- 해당 Domain으로 https 접속이 잘되고, 80 으로 접속한다면, 443 Redirect 된다.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/target-group-attributes: stickiness.enabled=true,stickiness.lb_cookie.duration_seconds=60
    # SSL Settings
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: 'ACM으로 생성된 Certificate ARN'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```






















