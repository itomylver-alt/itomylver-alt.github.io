---
title: "搭建Kubernetes Wordpress MySQL "
date: 2026-03-16
categories: [linux2]
layout: default
tags: [Docker,WordPress,Kubernetes,Nginx]
---
## 架构
浏览器 --> Ingress --> WordPress Service --> WordPress Pod --> MySQL Service --> MySQL Pod --> PersistentVolume(数据存储)

使用Minikube搭建本地集群 

## 安装
首先安装kubectl
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

添加执行权限
chmod +x kubectl

移动到系统路径
sudo mv kubectl /usr/local/bin/

验证
kubectl version --client


安装minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube

验证
minikube version

确保我们有docker

## 测试
启动minikube 
minikube start --driver=docker
如果出现 Done! kubectl is now configured to use "minikube" 那就成功

kubectl get nodes 查看是否部署成功

创建nginx deployment

#### 创建nginx
kubectl create deployment nginx-test --image=nginx

然后我们看kubectl get deploymentes 

kubectl get pods
类似nginx-test-7f9c6db8f

#### 进入容器
kubectl exec -it nginx-test-7f9c6db8f -- /bin/bash

ls /usr/share/nginx/html/
可以看到index.html的存放路径

##### 暴露服务
kubectl expose deployment nginx-test --type=NodePort --port=80

查看
kubectl get svc

#### 访问应用
minikube service nginx-test
浏览器会自动打开nginx页面


#### 学习扩容
kubectl scale deployment nginx-test --replicas=3
因为每个pod 都有自己的ip地址 所以端口不会冲突
我遇到的疑问

查看
kubectl get pods

#### 删除 pod 测试自动恢复
kubectl delete pod nginx-test-xxxxx
查看
kubectl get pods


## 首先创建 MySQL 密码
##### 创建Secret
 kubectl create secret generic mysql-pass --from-literal=password=123456
查看
  kubectl get secrets

## 创建MySQL Deployment
  #### 创建mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306

部署
 kubectl apply -f mysql-deployment.yaml
查看
kubectl get pods

## 创建MySQL Service

创建 mysql-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
spec:
  ports:
  - port: 3306
  selector:
    app: wordpress
    tier: mysql


应用
kubectl apply -f mysql-service.yaml

查看
kubectl get svc

## 创建WordPress Deployment
#### 创建 wordpress.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  replicas: 1
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - name: wordpress
        image: wordpress:latest
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80

部署
     kubectl apply -f wordpress.yaml

## 创建WordPress Service
kubectl expose deployment wordpress --type=NodePort --port=80

查看 
kubectl get svc


## 访问WordPress

minikube service wordpress


在部署的时候我遇到了问题
WordPress 连不上MySQL

查看容器日志
kubectl logs wordpress-xx

确认MySQL 是否运行
kubectl get pods
看到确实实在running

检查service 是否有pod
kubectl describe svc wordpress

看到pod 一直是containercreating   这个是正常   等他创建好  就Running了
但是并没有解决我的问题

我们确认一下集群的状态 minikube status

我之前遇到过 虚拟机没有磁盘空间了 导致docker 的Promethus 没有办法启动 
这次 看一下磁盘空间 df -h   还有很多 啊  我之前加了30GB

我们查一下是不是MySQL 的名字或密码错了
看wordpress的环境变量
kubectl describe pod wordpress-xxxxxx-xxxx
kubectl get svc
看HOST 和PASSWORD 是否一致
kubectl get secrets 看了一下

WordPress Deployment 看到密码 是来自mysql-pass
kubectl get secret mysql-pass -o yaml
可以看到一段base64加密的密码
我们直接 echo MWYyZDFlMmU2N2Rm | base64 -d
看一下 密码和 wordpress 用的一样啊

我又把wordpress pod 删除
kubectl delete pod wordpress-xxxxxx-xxxx
用浏览器打开
minikube service wordpress

还是Error establishing a database connection

我又进入wordpress的容器里面
kubectl exec -it wordpress-xxxx-xxxx -- /bin/bash
删掉wordpress的配置文件
rm /var/www/html/wp-config.php
然后删掉Pod
kubectl delete pod -l app=wordpress
让他重新生成
然后minikube service wordpress
还是没成功

我想进wordpess 的pod 看能不能ping通mysql 的pod
但是pod 没有ping
kubectl exec -it wordpress-xxx-xxx -- ping wordpress-mysql  不行



去网上找了 
cat < /dev/null > /dev/tcp/wordpress-mysql/3306
没有报错

启动一个带工具的pod
kubectl run debug --rm -it --image =busybox --restart=Never -- sh

DNS :
		nslookup wordpress-mysql
测试端口：
		nc -zv wordpress-mysql 3306
		
但是网络没有问题一切正常

还有mysql的用户权限问题
但是mysql  root 的权限是%  


没问题啊

进入MySQL Pod
kubectl exec -it wordpress-mysql-xxxx-xxx -- mysql -u root -p
密码 123456

show databases;

没有wordpress .....

然后我创建了 create database wordpress;

重启wordpress pod

kubectl delete pod -l app=wordpress

minikube service wordpress 
成功！！












