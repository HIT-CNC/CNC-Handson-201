# 201-1 Deploy a simple configuration on K8s

## 1. Login to the bastion server

Using your terminal emulator such as WSL, putty or ConEmu, login to the server

## 2. Make sure you have the .kube/config to connect to the Kuberentes API server

```
$ ls -l .kube/config
```

## 3. Try kubectl; the CLI tool for Kuberetes

```
$ kubectl get node
```
