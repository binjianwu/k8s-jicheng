### 部署gitlab

1. 创建数据库

这里使用已有数据库，所以首先创建用户和数据库。
```
# 创建数据库
create schema gitlabhq_production default character set utf8;
# 创建用户，赋予密码
CREATE USER gitlab@'%' IDENTIFIED BY 'password';
CREATE USER gitlab@'localhost' IDENTIFIED BY 'password';
# 把数据库gitlabhq_production的全部操作权限赋给用户gitlab
GRANT ALL PRIVILEGES ON gitlabhq_production.* TO gitlab@'%';
GRANT ALL PRIVILEGES ON gitlabhq_production.* TO gitlab@'localhost';

flush privileges;
```

2. 编写`gitlab.yaml`文件
```
kind: Namespace
apiVersion: v1
metadata:
    name: gitlab
---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: redis
  namespace: gitlab
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: registry.saas.hand-china.com/tools/redis:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 6379
          name: redis
        volumeMounts:
        - name: redis-home
          mountPath: /var/lib/redis:Z
      volumes:
         - name: redis-home
           hostPath:
            path: /srv/docker/gitlab/redis

---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: redis
  name: redis
  namespace: gitlab
spec:
  type: ClusterIP
  ports:
  - port: 6379
    protocol: TCP
  selector:
    app: redis

---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: gitlab
  namespace: gitlab
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      containers:
      - name: gitlab
        image: registry.saas.hand-china.com/tools/gitlab:8.15.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        - containerPort: 22
        volumeMounts:
        - name: gitlab-home
          mountPath: /home/git/data:Z
        env:
        - name: DEBUG
          value: "false"
        - name: DB_ADAPTER
          value: mysql2
        - name: DB_HOST
          value: mysql-mdm.db.svc.cluster.local
        - name: DB_PORT
          value: "3306"
        - name: DB_USER
          value: gitlab
        - name: DB_PASS
          value: password
        - name: DB_NAME
          value: gitlabhq_production
        - name: REDIS_HOST
          value: redis.gitlab.svc.cluster.local
        - name: REDIS_PORT
          value: "6379"
        - name: TZ
          value: "Asia/Kolkata"
        - name: GITLAB_TIMEZONE
          value: Kolkata
        - name: GITLAB_HTTPS
          value: "false"
        - name: SSL_SELF_SIGNED
          value: "false"
        - name: GITLAB_HOST
          value: "192.168.56.11"
        - name: GITLAB_PORT
          value: "30080"
        - name: GITLAB_SSH_PORT
          value: "30022"
        - name: GITLAB_RELATIVE_URL_ROOT
          value: 
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: ong-and-random-alphanumeric-string
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: long-and-random-alphanumeric-string
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: long-and-random-alphanumeric-string
        - name: GITLAB_ROOT_PASSWORD
          value: 
        - name: GITLAB_ROOT_EMAIL
          value: 
        - name: GITLAB_NOTIFY_ON_BROKEN_BUILDS
          value: "true"
        - name: GITLAB_NOTIFY_PUSHER
          value: "false"
        - name: GITLAB_EMAIL
          value: notifications@example.com
        - name: GITLAB_EMAIL_REPLY_TO
          value: noreply@example.com
        - name: GITLAB_INCOMING_EMAIL_ADDRESS
          value: reply@example.com
      volumes:
         - name: gitlab-home
           hostPath:
            path: /srv/docker/gitlab/gitlab

---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: gitlab
  name: gitlab
  namespace: gitlab
spec:
  type: NodePort
  ports:
  - port: 30080
    targetPort: 80
    protocol: TCP
    nodePort: 30080
    name: web
  - port: 30022
    targetPort: 22
    protocol: TCP
    nodePort: 30022
    name: ssh
  selector:
    app: gitlab
```
> 注意:这里`DB_HOST`和`REDIS_HOST`使用dns名字访问。

3. 创建
```
# kubectl get po -n gitlab
```
4. 访问

```
# http://host_address:30080
```