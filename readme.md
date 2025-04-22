# k8s Cluster Setup

## 01. 사전 준비

- VMware Workstation : 가상 머신 생성, Ubuntu 24.04 설치
- Kubernetes 버전 : 1.32
- K8S 설치 공식 문서 : https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

## 02. Node Info

![node_info.drawio](https://github.com/revenge1005/k8s-cluster-setup/blob/main/node_info.drawio.png)

## 03. 사전 준비

### A) k8s-master, k8s-worker1, k8s-worker2의 system 구성 정보 확인 및 설정

```bash

```

### B) IPv4를 포워딩하여 iptables가 bridge된 traffic을 보게 하기

```bash

```

## 04. Kubernetes Cluster Setup Guide

- [1. Installing Kubernetes with Docker Engine Runtime]()
- [2. Installing Kubernetes with containerd Runtime]()