# 애플리케이션을 쿠버네티스에 배포하기 (yaml 파일 사용)

지금까지 애플리케이션을 작성하고, 해당 애플리케이션을 컨테이너화를 했습니다.

이번 단계는 애플리케이션을 쿠버네티스에 배포하는 단계입니다.

>애플리케이션의 배포는 CI/CD Pipeline을 통해 쉽게 쿠버네티스에 배포 가능합니다.   
>이번 세션에서는 Yaml 파일을 통한 배포를 하도록 하겠습니다.

## 사전 요구사항
- docker login
- Kubernetes Cluster ([OpenShift](https://developers.redhat.com/products/openshift/download), [minikube](https://minikube.sigs.k8s.io/docs/start/), 등)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)


## 컨테이너 이미지를 컨테이너 저장소에 넣기

쿠버네티스는 컨테이너 저장소에서 컨테이너를 클러스터에 배포를 합니다. 그래서 이 전 세션에서 만들었던 이미지를 컨테이너 저장소에 넣는 작업이 필요합니다.

컨테이너 저장소는 여러가지가 있습니다. docker.io 나 registry.access.redhat.com 같이 public 저장소도 있고 자체 네트워크에서 구성하는 private 저장소도 있습니다.  본 세션에서는 docker.io를 사용하기 위해서 docker.io에 로그인이 필요합니다.

>docker.io에서 아이디를 새로 만들 수 있습니다. 아이디를 새로 만드셨다면 반드시 email을 확인하셔서 검증하십시오.
~~~sh
docker login
~~~

그리고 이 전 세션에서 만들었던 컨테이너 이미지를 push 합니다.
~~~sh
docker push $DOCKER_USER/member-app
~~~~

아래와 같이 진행되어 완료되게 됩니다.
~~~sh
Using default tag: latest
The push refers to repository [docker.io/jonggyoukim/member-app]
dc6a1c37fcb2: Pushed 
e9354aaf0747: Pushed 
3d7859fc32bc: Mounted from library/node 
d702bac94b3c: Mounted from library/node
7e84055b0824: Mounted from library/node 
08592983d1e9: Mounted from library/node 
cdc9dae211b4: Mounted from library/node 
7095af798ace: Mounted from library/node 
fe6a4fdbedc0: Mounted from library/node 
e4d0e810d54a: Mounted from library/node 
4e006334a6fd: Mounted from library/node 
latest: digest: sha256:e110f145b2fedc0f2bb5960032c67f61c3c10da0780b465bb2135f4836e5a2df size: 2635
~~~


브라우저를 통해 docker.io/jonggyoukim 을 살펴보면 다음과 같습니다.

![](./images/docker-io-1.png)


## MySQL 배포하기

Filename: deployment-mysql.yaml
~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-mysql
  labels:
    app: demo-mysql
spec:
  selector:
    matchLabels:
      app: demo-mysql

  template:
    metadata:
      labels:
        app: demo-mysql
    spec:
      containers:
      - name: demo-mysql
        image: mysql:5.6
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: mypassword
        ports:
        - containerPort: 3306
~~~


~~~sh
$ kubectl apply -f deployment-mysql.yaml

deployment.apps/demo-mysql created
~~~
~~~sh
$ kubectl get deployment demo-mysql

NAME         READY   UP-TO-DATE   AVAILABLE   AGE
demo-mysql   1/1     1            1           4m32s
~~~

~~~sh
$ kubectl get pod --selector=app=demo-mysql

NAME                          READY   STATUS    RESTARTS   AGE
demo-mysql-66f4dc5774-w6phm   1/1     Running   0          2m5s
~~~


~~~
$ kubectl exec -it demo-mysql-66f4dc5774-w6phm -- /bin/sh

$
~~~

~~~
$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.6.51 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> CREATE USER 'test'@'%' IDENTIFIED BY 'Welcome1';
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql>     GRANT USAGE ON *.* TO 'test'@'%';
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql>     GRANT ALL PRIVILEGES ON *.* TO 'test'@'%';
Query OK, 0 rows affected (0.00 sec)

mysql>
mysql>     CREATE DATABASE sample DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
Query OK, 1 row affected (0.00 sec)

mysql>
mysql>     USE sample;
Database changed
mysql>
mysql>     CREATE TABLE IF NOT EXISTS `players` (
    ->     `id` int(5) NOT NULL AUTO_INCREMENT,
    ->     `first_name` varchar(255) NOT NULL,
    ->     `last_name` varchar(255) NOT NULL,
    ->     `position` varchar(255) NOT NULL,
    ->     `number` int(11) NOT NULL,
    ->     `user_name` varchar(20) NOT NULL,
    ->     PRIMARY KEY (`id`)
    ->     ) ENGINE=InnoDB  AUTO_INCREMENT=1;
Query OK, 0 rows affected (0.02 sec)

mysql> exit
~~~

## 애플리케이션 배포하기

`yaml` 디렉토리에 deployment.yaml 파일이 존재합니다. 이 파일은 docker.io에 저장한 jonggyoukim/member-app 를 쿠버네티스에 배포하도록 정의하는 파일입니다.

>File: yaml/deployment.yaml
~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app: demo-app
spec:
  selector:
    matchLabels:
      app: demo-app

  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: jonggyoukim/member-app  
        env:
        - name: MYSQL_SERVICE_HOST
          value: demo-mysql
        ports:
        - containerPort: 8080
          name: demo-app
~~~

- 배포하는 이미지는 `jonggyoukim/member-app`로 지정
- 환경변수로 MySQL에 접근하기 위한 MYSQL_SERVICE_HOST 변수에 대한 값으로 `demo-mysql`이 지정
- 포트는 8080 지정




