# 基于KubeSphere的Kubernetes生产实践之路-SpingCloud架构中间件的安装部署

大家好，我是老Z

本文接着上篇 **<<基于KubeSphere的Kubernetes生产实践之路-持久化存储之GlusterFS>>** 继续打造我们Kubernetes生产环境。

## 前提说明

1. 业务程序使用SpringCloud框架开发

2. 如果采用离线部署的方式，所有相关镜像需要提前push到镜像仓库，本文略过

3. 部署架构图
   
   <img title="" src="https://gitee.com/zdevops/res/raw/main/cloudnative/middleware.png">

4. 整体部署架构涉及的中间件包含以下组件
   
   - nacos
   
   - RocketMQ
   
   - Redis
   
   - xxl-job
   
   - skywalking
   
   - MySQL
   
   - Nginx

5. 部署架构说明
   
   - k8s集群之外采用了nginx作为网关代理，将外部流量引入k8s集群
   
   - k8s集群对外暴露服务使用了nodeport的方式，暂时没有引入Ingress(因为不会，后期学会了再引入)
   
   - 其他的中间件都是部署在k8s集群之内
   
   - 本文并没有写日志配置的相关内容，后续会有专门的文档介绍
   
   - 本文没有介绍业务模块的配置内容，相关内容会在**业务模块自动化发布**文档中专门介绍



## 网关代理安装配置

### 环境说明

- 操作系统 CentOS7.9

- Nginx 1.20.2的rpm包

### 部署步骤

```shell
# 下载rpm离线包，这里选择官方的最新稳定版1.20.2
[root@nginx-0 ~]# wget http:/nginx.org/packages/centos/7/x86_64/RPMS/nginx-1.20.2-1.el7.ngx.x86_64.rpm

# 安装
[root@nginx-0 ~]# rpm -ivh nginx-1.20.2-1.el7.ngx.x86_64.rpm
warning: nginx-1.20.2-1.el7.ngx.x86_64.rpm: Header V4 RSA/SHA1 Signature, key ID 7bd9bf62: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:nginx-1:1.20.2-1.el7.ngx         ################################# [100%]

# 查看已安装的nginx
[root@nginx-0 ~]# rpm -qa | grep nginx
nginx-1.20.2-1.el7.ngx.x86_64

# 根据需要修改/etc/nginx/nginx.conf（主要用于自定义nginx服务器的各种配置等）
[root@nginx-0 ~]# vi /etc/nginx/nginx.conf
示例配置见配置文件参考

# 根据需要修改/etc/nginx/conf.d/***.conf（主要用于自定义nginx的各种转发规则等）
[root@nginx-0 ~]# vi /etc/nginx/conf.d/***.conf
示例配置见配置文件参考

# 检测nginx配置文件
[root@nginx-0 ~]# nginx -t

# 启动nginx，并配置开机自启
[root@nginx-0 ~]# systemctl start nginx && systemctl enable nginx
```

### 配置文件参考

1. nginx.conf (/etc/nginx/nginx.conf)
   
   ```yaml
   user nginx;
   worker_processes auto;
   error_log /var/log/nginx/error.log;
   pid /run/nginx.pid;
   
   # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
   include /usr/share/nginx/modules/*.conf;
   
   events {
       worker_connections 10240;
   }
   
   http {
       server_tokens off;                                                #隐藏版本号
       client_header_timeout 60;                                        #客户端向服务端发送一个完整的 request header 的超时时间。如果客户端在指定时间内没有发送一个完整的 request header，Nginx 返回 HTTP 408(“Request timed out”)
       client_body_timeout 60;                                            #该指令设置请求正文即请求体（request body）的读超时时间。超时仅设置为两个连续读取操作之间的时间段，而不是整个请求主体的传输。如果客户端在此时间内未传输任何内容，请求将以408（请求超时）错误终止
       limit_conn_zone $binary_remote_addr zone=one:10m;                #限制可以存储多少个并发连接数（1m 可以储存 32000 个并发会话）
       limit_conn one 50;                                                #限制每个IP只能发起50个并发连接
       limit_rate 2000k;                                                #控制下载速度
       send_timeout 10;                                                #服务端向客户端传输数据的超时时间，单位（s）
       keepalive_timeout   65;                                            #每个TCP连接的超时时间
       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';
   
       log_format  debug  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for" '
                         '"debug-cors" $cors_origin "debug-origin" $http_origin ';
   
       access_log  /var/log/nginx/access.log  main;
   
       sendfile            on;                                            #sendfile是个比 read 和 write 更高性能的系统接口， 不过需要注意的是，sendfile 是将 in_fd 的内容发送到 out_fd 。而 in_fd 不能是 socket ， 也就是只能文件句柄。 所以当 Nginx 是一个静态文件服务器的时候，开启 SENDFILE 配置项能大大提高 Nginx 的性能.
       tcp_nopush          on;                                            #可以配置一次发送数据包的大小。也就是说，数据包累积到一定大小后就发送，tcp_nopush必须和sendfile配合使用.
       tcp_nodelay         on;                                            #会增加小包的数量，但是可以提高响应速度。在及时性高的通信场景中应该会有不错的效果
       types_hash_max_size 4096;                                        #nginx使用了一个散列表来保存MIME type与文件扩展名之间的映射，该参数就是指定该散列表桶的大小的
   
       include             /etc/nginx/mime.types;
       default_type        application/octet-stream;
       error_page  400 404 413 502 504  /index.html;
   
       #开启gzip压缩
       gzip  on;
       #不压缩临界值，大于1K的才压缩
       gzip_min_length 1k;
       #buffer
       gzip_buffers 4 16k;
       #用了反向代理的话，末端通信是HTTP/1.0,默认是HTTP/1.1
       #gzip_http_version 1.0;
       #压缩级别，1-10，数字越大压缩的越好，时间也越长
       gzip_comp_level 2;
       #进行压缩的文件类型，缺啥补啥
       gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascriptapplication/x-httpd-php image/jpeg image/gif image/png;
       #跟Squid等缓存服务有关，on的话会在Header里增加"Vary: Accept-Encoding"
       gzip_vary off;
       #IE6对Gzip不友好，不进行Gzip压缩
       gzip_disable "MSIE [1-6]\.";
   
       client_max_body_size 100m;                                    #nginx对上传文件大小的限制
       proxy_buffer_size  128k;                                    #Nginx使用该大小申请read_buf，即大小指定了 upstream header 最大长度，如果响应头超过了这个长度，Nginx会报upstream sent too big header错误，然后client收到的是502
       proxy_buffers   32 32k;                                        #设置存储被代理服务器响应的body所占用的buffer个数和每个buffer大小
       proxy_busy_buffers_size 128k;                                #proxy_busy_buffers_size不是独立的空间，他是proxy_buffers和proxy_buffer_size的一部分。nginx会在没有完全读完后端响应就开始向客户端传送数据，所以它会划出一部分busy状态的buffer来专门向客户端传送数据(建议为proxy_buffers中单个缓冲区的2倍)，然后它继续从后端取数据。proxy_busy_buffer_size参数用来设置处于busy状态的buffer有多大
   
       fastcgi_buffers 16 256k;                                    #设定用来读取从FastCGI服务器端收到的响应信息的缓冲区大小和缓冲区数量
       fastcgi_buffer_size 128k;                                    #Nginx FastCGI 的缓冲区大小，用来读取从FastCGI服务器端收到的第一部分响应信息的缓冲区大小
       fastcgi_busy_buffers_size 256k;                                #用于设置系统很忙时可以使用的 proxy_buffers 大小
   
       map $http_upgrade $connection_upgrade {  #开启websocket升级代理功能，可选
           default upgrade;
           '' close;
       }
   
       include /etc/nginx/conf.d/*.conf;
   
       server {                                                    #http（80）转https（443）配置                
           listen       80;
           listen       [::]:80;
           server_name  www.abc.com;
           rewrite ^(.*)$ https://${server_name}$1 permanent;
           #root         /usr/share/nginx/html;
   
           # Load configuration files for the default server block.
           include /etc/nginx/default.d/*.conf;
   
           error_page 404 /404.html;
           location = /404.html {
           }
   
           error_page 500 502 503 504 /50x.html;
           location = /50x.html {
           }
       }
   }
   ```

2. default.conf(/etc/nginx/conf.d/default.conf)
   
   配置文件里只是一个后端模块的配置，请根据实际情况增加对应的location
   
   ```yaml
   server {
   listen 80;
   　　server_name www.abcd.com;
      access_log  /var/log/nginx/abc.access.log  main;                #自定义专属日志文件
   　　
   　　location / {
   　　　　root /usr/share/nginx/dist;
   　　　　index  index.html index.htm;
   　　}
   
       location ^~ /api/ {                                            #转发示例
           proxy_pass http://192.168.1.1:8003/;
           proxy_redirect off;
           proxy_set_header Host $host;
           proxy_set_header REMOTE-HOST $remote_addr;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```



## k8s部署MySQL5.7

Nacos需要使用MySQL存储配置数据，由于使用量不大，因此没有考虑高可用部署，也可以采用已有的MySQL数据库。

### 部署步骤

#### 创建MySQL相关部署文件

1. 创建mysql存储pvc yaml文件

```shell
[root@k8s-master-0 ~]# vi mysql-pvc.yaml
```

- mysql-pvc.yaml

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-pvc
  namespace: test                                          #注意修改命名空间
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: glusterfs
```

2. 创建mysql配置configmap yaml文件

```shell
[root@k8s-master-0 ~]# vi mysql-config.yaml
```

- mysql-config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-cm
  namespace: test                                        #注意修改命名空间
data:
  mysqld.cnf: |
    [mysqld]
    pid-file        = /var/run/mysqld/mysqld.pid
    socket          = /var/run/mysqld/mysqld.sock
    datadir         = /var/lib/mysql
    bind-address    = 0.0.0.0
    port = 3306
    log-bin = mysql-bin
    server-id = 1
```

3. 创建mysql服务service yaml文件

```shell
[root@k8s-master-0 ~]# vi mysql-service.yaml
```

- mysql-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: test                                            #注意修改命名空间
spec:
  type: NodePort
  ports:
  - port: 3306
    targetPort: 3306
    nodePort: 30850
  selector:
    app: mysql
```

4. 创建mysql配置副本Deployment yaml文件

```shell
[root@k8s-master-0 ~]# vi mysql.yaml
```

- mysql.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: test                                                    #注意修改命名空间
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7.32
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "root@my.123"                                         #数据库root的密码
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
        volumeMounts:
        - name: mysql-config
          mountPath: /etc/mysql/mysql.conf.d/mysqld.cnf
          subPath: mysqld.cnf
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
      - name: mysql-config
        configMap:
          name: mysql-cm
```

#### 部署MySQL

```shell
[root@k8s-master01 ~]# kubectl apply -f mysql-pvc.yaml
[root@k8s-master01 ~]# kubectl apply -f mysql-config.yaml
[root@k8s-master01 ~]# kubectl apply -f mysql-service.yaml
[root@k8s-master01 ~]# kubectl apply -f mysql.yaml
```

#### 验证MySQL

```shell
[root@k8s-master01 ~]# kubectl get pods -n test
NAME                     READY   STATUS    RESTARTS   AGE
mysql-589dcf6597-5ps6x   1/1     Running   0          8m3s

# 进入pod登录验证
[root@k8s-master01 ~]# kubectl exec -it mysql-589dcf6597-5ps6x /bin/bash -n test
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mysql-589dcf6597-5ps6x:/# mysql -u root -p
Enter password:

mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'P@ssword-123';        #修改root密码
Query OK, 0 rows affected (0.00 sec)
```



## k8s部署Nacos集群

### 部署步骤

#### 拉取项目代码

```shell
#拉取nacos代码
[root@k8s-master-0 ~]# git clone https://github.com/nacos-group/nacos-k8s.git

#拉取nacos的初始化数据库sql文件
[root@k8s-master-0 ~]# wget https://github.com/alibaba/nacos/blob/develop/distribution/conf/nacos-mysql.sql
```

#### 创建nacos所需数据库

```shell
# 登录数据库
[root@k8s-master01 mysql]# kubectl exec -it mysql-589dcf6597-5ps6x /bin/bash -n test
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mysql-589dcf6597-5ps6x:/# mysql -u root -p
Enter password:
# 创建数据库
mysql> CREATE DATABASE  IF NOT EXISTS `nacos_dev` DEFAULT CHARACTER SET utf8mb4 DEFAULT COLLATE utf8mb4_unicode_ci;

# 创建nacos用户
mysql> GRANT ALL PRIVILEGES ON  nacos.* to nacos@'%' IDENTIFIED BY 'nacos';

# 刷新权限
mysql> FLUSH PRIVILEGES;

# 导入nacos数据库
mysql> use nacos;
Database changed

# 查看mysql的连接端口
[root@k8s-master01 mysql]# kubectl get svc -n test
NAME                                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
mysql                                                    NodePort    10.233.20.110   <none>        3306:30850/TCP   47m

# 使用mysql连接工具连接数据库并导入上方sql
# 因为mysql service的类型为nodeport,所以连接地址为任一k8s集群的节点ip地址，端口为30850

# 回到命令行查看nacos数据表
mysql> show tables;
+----------------------+
| Tables_in_nacos      |
+----------------------+
| config_info          |
| config_info_aggr     |
| config_info_beta     |
| config_info_tag      |
| config_tags_relation |
| group_capacity       |
| his_config_info      |
| permissions          |
| roles                |
| tenant_capacity      |
| tenant_info          |
| users                |
+----------------------+
12 rows in set (0.00 sec)
```

#### 修改nacos部署yaml文件

```shell
#进入项目目录
[root@k8s-master-0 ~]# cd nacos-k8s/deploy/nacos
[root@k8s-master-0 ~]# cp nacos-pvc-nfs.yaml nacos.yaml

# 修改部署文件
[root@k8s-master-0 nacos]# vi nacos.yaml
```

- nacos.yaml

```yaml
---                         #修改8848端口为nodeport形式,以便集群外部访问
apiVersion: v1
kind: Service
metadata:
  name: nacos-server
  namespace: test                            #增加命名空间
  labels:
    app: nacos
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  type: NodePort
  ports:
    - port: 8848
      name: server
      targetPort: 8848
      nodePort: 30848
  selector:
    app: nacos
---
apiVersion: v1
kind: Service
metadata:
  name: nacos-headless
  namespace: test                            #增加命名空间
  labels:
    app: nacos
  annotations:
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  ports:
    - port: 8848
      name: server
      targetPort: 8848
    - port: 9848
      name: client-rpc
      targetPort: 9848
    - port: 9849
      name: raft-rpc
      targetPort: 9849
    ## 兼容1.4.x版本的选举端口
    - port: 7848
      name: old-raft-rpc
      targetPort: 7848
  clusterIP: None
  selector:
    app: nacos
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nacos-cm
  namespace: test                            #增加命名空间
data:
  mysql.host: "mysql.test"                 #增加mysql连接地址
  mysql.db.name: "nacos_devtest"
  mysql.port: "3306"
  mysql.user: "nacos"
  mysql.password: "nacos"
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nacos
  namespace: test                            #增加命名空间
spec:
  serviceName: nacos-headless
  replicas: 3
  template:
    metadata:
      labels:
        app: nacos
      annotations:
        pod.alpha.kubernetes.io/initialized: "true"
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                      - nacos
              topologyKey: "kubernetes.io/hostname"
      initContainers:
        - name: peer-finder-plugin-install
          image: nacos/nacos-peer-finder-plugin:1.1
          imagePullPolicy: Always
          volumeMounts:
            - mountPath: /home/nacos/plugins/peer-finder
              name: data
              subPath: peer-finder
      containers:
        - name: nacos
          imagePullPolicy: Always
          image: nacos/nacos-server:latest
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
          ports:
            - containerPort: 8848
              name: client-port
            - containerPort: 9848
              name: client-rpc
            - containerPort: 9849
              name: raft-rpc
            - containerPort: 7848
              name: old-raft-rpc
          env:
            - name: NACOS_REPLICAS
              value: "3"
            - name: SERVICE_NAME
              value: "nacos-headless"
            - name: DOMAIN_NAME
              value: "cluster.local"     #修改k8s集群地址，可通过cat /etc/kubernetes/kubelet.conf查看对应字段：contexts/- context/cluster
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.namespace
            - name: MYSQL_SERVICE_HOST              #增加获取mysql连接地址的环境变量
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.host
            - name: MYSQL_SERVICE_DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.db.name
            - name: MYSQL_SERVICE_PORT
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.port
            - name: MYSQL_SERVICE_USER
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.user
            - name: MYSQL_SERVICE_PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: nacos-cm
                  key: mysql.password
            - name: NACOS_SERVER_PORT
              value: "8848"
            - name: NACOS_APPLICATION_PORT
              value: "8848"
            - name: PREFER_HOST_MODE
              value: "hostname"
          volumeMounts:
            - name: data
              mountPath: /home/nacos/plugins/peer-finder
              subPath: peer-finder
            - name: data
              mountPath: /home/nacos/data
              subPath: data
            - name: data
              mountPath: /home/nacos/logs
              subPath: logs
  volumeClaimTemplates:
    - metadata:
        name: data
        namespace: test                                #增加命名空间
        annotations:
          volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
      spec:
        accessModes: [ "ReadWriteMany" ]
        storageClassName: "glusterfs"                    #增加storageClass选择器，以便自动创建pvc
        resources:
          requests:
            storage: 20Gi
  selector:
    matchLabels:
      app: nacos
```

#### 部署集群

```shell
[root@k8s-master-0 ~]# kubectl apply  -f  nacos.yaml
```

#### 验证集群

```shell
# 查看nacos pod
[root@k8s-master-0 ~]# kubectl get pod -n test | grep nacos
nacos-0                                               1/1     Running   0          120d
nacos-1                                               1/1     Running   0          120d
nacos-2                                               1/1     Running   0          120d

# 查看nacos svc
[root@k8s-master01 ~]# kubectl get svc -n test | grep nacos
NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
nacos-headless                     ClusterIP   None            <none>        8848/TCP,9848/TCP,9849/TCP,7848/TCP   216d
nacos-server                       NodePort    10.233.20.35    <none>        8848:30848/TCP                        213d

# 访问任意节点ip+30848（nodeport端口）进行验证 http://ip:30848/nacos
默认用户名/密码 ：nacos/nacos
```



## k8s部署Redis集群

Redis采用三主三从集群模式部署

### 部署步骤

#### 创建redis副本文件

```shell
[root@k8s-master-0 ~]# vi redis-statefulset.yaml
```

- redis-statefulset.yaml

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-cm
  namespace: test
data:
  redis-conf: |
    appendonly yes                                                        #开启AOF模式
    protected-mode no                                                    #关闭protected-mode模式，此时外部网络可以直接访问
    cluster-enabled yes                                                    #开启集群模式
    cluster-config-file /data/nodes.conf                                #Redis集群节点的集群配置文件
    cluster-node-timeout 5000                                            #指节点在失败状态下必须不可到达的毫秒数。大多数其他内部时间限制是节点超时的倍数
    dir /data                                                            #数据存储目录
    port 6379
    requirepass redis@123.com                                            #redis密码，自定义
    masterauth redis@123.com                                            #如果master是密码保护的，在启动复制同步进程之前，可以告诉奴隶进行身份验证，否则主人将拒绝奴隶请求。
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: test
  labels:
    app: redis
spec:
  serviceName: redis-headless
  replicas: 6
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - redis
              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: dockerhub.test.com:18443/library/redis:6.2.5
        command:
          - "redis-server"
        args:
          - "/etc/redis/redis.conf"
          - "--protected-mode"
          - "no"
          - "--cluster-announce-ip"
          - "$(POD_IP)"
        env:
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        ports:
            - name: redis
              containerPort: 6379
              protocol: "TCP"
            - name: cluster
              containerPort: 16379
              protocol: "TCP"
        volumeMounts:
          - name: redis-conf
            mountPath: /etc/redis        
          - name: redis-data
            mountPath: /data
      volumes:
      - name: redis-conf
        configMap:
          name: redis-cluster-cm
          items:
            - key: redis-conf
              path: redis.conf
  volumeClaimTemplates:
    - metadata:
        name: redis-data
        namespace: test
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "glusterfs"                                        #注意修改为自己的storageClass
        resources:
          requests:
            storage: 10Gi                                                    #pvc容量，自定义

---
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: test
  labels:
    app: redis
spec:
  type: NodePort
  ports:
    - name: redis-port
      port: 6379
      targetPort: 6379
      nodePort: 30849                                                        #nodeport端口自定义
  selector:
    app: redis
```

#### 部署并配置集群

```shell
# 部署
[root@k8s-master-0 ~]# kubectl apply -f redis-statefulset.yaml

# 配置集群
# 自动配置3个master,3个slave节点的集群，-a指定密码
[root@k8s-master-0 ~]# kubectl exec -it redis-0 -n test -- redis-cli -a redis@123.com --cluster create --cluster-replicas 1 $(kubectl get pods -n test -l app=redis -o jsonpath='{range.items[*]}{.status.podIP}:6379 {end}')
```

#### 验证集群

```shell
# 对redis集群进行验证
[root@k8s-master-0 ~]# kubectl exec -it redis-0 -n test -- redis-cli -a redis@123.com --cluster check $(kubectl get pods -n test -l app=redis -o jsonpath='{range.items[0]}{.status.podIP}:6379{end}')
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
10.233.67.76:6379 (b8e966ed...) -> 286 keys | 5461 slots | 1 slaves.
10.233.94.207:6379 (31d925a7...) -> 263 keys | 5462 slots | 1 slaves.
10.233.98.106:6379 (11b42330...) -> 275 keys | 5461 slots | 1 slaves.
[OK] 824 keys in 3 masters.
0.05 keys per slot on average.
>>> Performing Cluster Check (using node 10.233.67.76:6379)
M: b8e966ed2e00d2c9fb24ebdd409fd7eef90cbb11 10.233.67.76:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: 35d46c8a708f234b73d647d1a800e52a620f2fbd 10.233.82.103:6379
   slots: (0 slots) slave
   replicates 31d925a76d8276ec6b1735a65b3e8d238ca5b63f
S: 8bbb3e01a5d9087327ff5ea2ee57e87c772c5ba9 10.233.94.205:6379
   slots: (0 slots) slave
   replicates 11b42330416a1fe3da01ff696b573e35f95be0c6
M: 31d925a76d8276ec6b1735a65b3e8d238ca5b63f 10.233.94.207:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
M: 11b42330416a1fe3da01ff696b573e35f95be0c6 10.233.98.106:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 121cdf362ae961cb18df31588df56e2e0cd42f10 10.233.123.249:6379
   slots: (0 slots) slave
   replicates b8e966ed2e00d2c9fb24ebdd409fd7eef90cbb11
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

[root@k8s-master-0 ~]# kubectl get pod -n test | grep redis
redis-0                                               1/1     Running   0          130d
redis-1                                               1/1     Running   0          130d
redis-2                                               1/1     Running   0          130d
redis-3                                               1/1     Running   0          130d
redis-4                                               1/1     Running   0          130d
redis-5                                               1/1     Running   0          130d
```



## k8s部署RocketMQ集群

- 为了实现快速和简单的部署，RocketMQ的部署采用了官方提供的operator

- 但是官方的部署使用后期发现诸多不便之处，也可能是我没玩明白，还需要深入研究
  
  - pod如果重建的话ui可能会连不上集群
  
  - pod重建后集群发现也出现过问题

### 部署步骤

#### 部署RocketMQ相关组件的crd资源

```shell
# 拉取rocketmq部署文件
[root@k8s-master-0 ~]# git clone -b 0.2.1 https://github.com/apache/rocketmq-operator.git
[root@k8s-master-0 ~]# cd rocketmq-operator

# 查看crd部署脚本
[root@k8s-master-0 rocketmq-operator]# cat install-operator.sh
#!/bin/bash

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

kubectl create -f deploy/crds/rocketmq_v1alpha1_broker_crd.yaml
kubectl create -f deploy/crds/rocketmq_v1alpha1_nameservice_crd.yaml
kubectl create -f deploy/crds/rocketmq_v1alpha1_topictransfer_crd.yaml
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml
# kubectl create -f example/rocketmq_v1alpha1_rocketmq_cluster.yaml


# 为部署脚本中的所有yaml文件增加命名空间
[root@k8s-master-0 rocketmq-operator]# vi deploy/crds/rocketmq_v1alpha1_broker_crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: brokers.rocketmq.apache.org
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/crds/rocketmq_v1alpha1_nameservice_crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: nameservices.rocketmq.apache.org
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/crds/rocketmq_v1alpha1_consoles_crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: consoles.rocketmq.apache.org
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/crds/rocketmq_v1alpha1_topictransfer_crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: topictransfers.rocketmq.apache.org
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/service_account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rocketmq-operator
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  creationTimestamp: null
  name: rocketmq-operator
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/role_binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rocketmq-operator
  namespace: test

[root@k8s-master-0 rocketmq-operator]# vi deploy/operator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rocketmq-operator
  namespace: test

# 创建rocketmq Operator
[root@k8s-master-0 rocketmq-operator]# sh install-operator.sh
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/brokers.rocketmq.apache.org created
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/nameservices.rocketmq.apache.org created
Warning: apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition
customresourcedefinition.apiextensions.k8s.io/topictransfers.rocketmq.apache.org created
serviceaccount/rocketmq-operator created
role.rbac.authorization.k8s.io/rocketmq-operator created
rolebinding.rbac.authorization.k8s.io/rocketmq-operator created
deployment.apps/rocketmq-operator created

# 查看rocketmq Operator
[root@k8s-master01 rocketmq-operator]# kubectl get pod -n test
NAME                               READY   STATUS    RESTARTS   AGE
rocketmq-operator-867c4955-dhgzh   1/1     Running   0          7m40s
```

#### 配置RocketMQ集群部署yaml文件

```shell
[root@k8s-master-0 rocketmq-operator]# vi example/rocketmq_v1alpha1_rocketmq_cluster.yaml
```

- rocketmq_v1alpha1_rocketmq_cluster.yaml

```yaml
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: ConfigMap
metadata:
  name: broker-config
  namespace: test                                            #添加命名空间
data:
  # BROKER_MEM sets the broker JVM, if set to "" then Xms = Xmx = max(min(1/2 ram, 1024MB), min(1/4 ram, 8GB))
  BROKER_MEM: " -Xms2g -Xmx2g -Xmn1g "
  broker-common.conf: |
    # brokerClusterName, brokerName, brokerId are automatically generated by the operator and do not set it manually!!!
    deleteWhen=04
    fileReservedTime=48
    flushDiskType=ASYNC_FLUSH
    # set brokerRole to ASYNC_MASTER or SYNC_MASTER. DO NOT set to SLAVE because the replica instance will automatically be set!!!
    brokerRole=ASYNC_MASTER

---
apiVersion: rocketmq.apache.org/v1alpha1
kind: Broker
metadata:
  # name of broker cluster
  name: broker
  namespace: test                                            #添加命名空间
spec:
  # size is the number of the broker cluster, each broker cluster contains a master broker and [replicaPerGroup] replica brokers.
  size: 1
  # nameServers is the [ip:port] list of name service
  nameServers: ""                                            #无需填写自动获取
  # replicaPerGroup is the number of each broker cluster
  replicaPerGroup: 1
  # brokerImage is the customized docker image repo of the RocketMQ broker
  brokerImage: apacherocketmq/rocketmq-broker:4.5.0-alpine-operator-0.3.0
  # imagePullPolicy is the image pull policy
  imagePullPolicy: Always
  # resources describes the compute resource requirements and limits
  resources:
    requests:
      memory: "2048Mi"
      cpu: "250m"
    limits:
      memory: "12288Mi"
      cpu: "500m"
  # allowRestart defines whether allow pod restart
  allowRestart: true
  # storageMode can be EmptyDir, HostPath, StorageClass
  storageMode: StorageClass
  # hostPath is the local path to store data
  hostPath: /data/rocketmq/broker
  # scalePodName is [Broker name]-[broker group number]-master-0
  scalePodName: broker-0-master-0
  # env defines custom env, e.g. BROKER_MEM
  env:
    - name: BROKER_MEM
      valueFrom:
        configMapKeyRef:
          name: broker-config
          key: BROKER_MEM
  # volumes defines the broker.conf
  volumes:
    - name: broker-config
      configMap:
        name: broker-config
        items:
          - key: broker-common.conf
            path: broker-common.conf
  # volumeClaimTemplates defines the storageClass
  volumeClaimTemplates:
    - metadata:
        name: broker-storage
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: "glusterfs"                            #修改为自己的storageClass
        resources:
          requests:
            storage: 8Gi                                        #可自定义pvc容量
---
apiVersion: rocketmq.apache.org/v1alpha1
kind: NameService
metadata:
  name: name-service
  namespace: test                                                #添加命名空间
spec:
  # size is the the name service instance number of the name service cluster
  size: 1
  # nameServiceImage is the customized docker image repo of the RocketMQ name service
  nameServiceImage: apacherocketmq/rocketmq-nameserver:4.5.0-alpine-operator-0.3.0
  # imagePullPolicy is the image pull policy
  imagePullPolicy: Always
  # hostNetwork can be true or false
  hostNetwork: true
  #  Set DNS policy for the pod.
  #  Defaults to "ClusterFirst".
  #  Valid values are 'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'.
  #  DNS parameters given in DNSConfig will be merged with the policy selected with DNSPolicy.
  #  To have DNS options set along with hostNetwork, you have to specify DNS policy
  #  explicitly to 'ClusterFirstWithHostNet'.
  dnsPolicy: ClusterFirstWithHostNet
  # resources describes the compute resource requirements and limits
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1024Mi"
      cpu: "500m"
  # storageMode can be EmptyDir, HostPath, StorageClass
  storageMode: StorageClass
  # hostPath is the local path to store data
  hostPath: /data/rocketmq/nameserver
  # volumeClaimTemplates defines the storageClass
  volumeClaimTemplates:
    - metadata:
        name: namesrv-storage
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: rocketmq-storage
        storageClassName: "glusterfs"                                #修改为自己的storageClass
        resources:
          requests:
            storage: 1Gi                                            #可自定义pvc容量

---
apiVersion: rocketmq.apache.org/v1alpha1
kind: Console
metadata:
  name: console
  namespace: test                                                    #添加命名空间
spec:
  # nameServers is the [ip:port] list of name service
  nameServers: ""
  # consoleDeployment define the console deployment
  consoleDeployment:
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: rocketmq-console
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: rocketmq-console
      template:
        metadata:
          labels:
            app: rocketmq-console
        spec:
          containers:
            - name: console
              image: apacherocketmq/rocketmq-console:2.0.0
              ports:
                - containerPort: 8080
```

#### 配置RocketMQ集群service yaml文件

```shell
[root@k8s-master-0 rocketmq-operator]# vi example/rocketmq_v1alpha1_cluster_service.yaml
```

- rocketmq_v1alpha1_cluster_service.yaml

```shell
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Service
metadata:
  name: console-service
  namespace: test                                    #修改命名空间
  labels:
    app: rocketmq-console
spec:
  type: NodePort
  selector:
    app: rocketmq-console
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      nodePort: 30849                                #注意修改nodeport端口
#---
#apiVersion: v1                                        #如果集群外的服务需要使用rockermq可以取消此service注释
#kind: Service
#metadata:
#  name: name-server-service
#  namespace: test
#spec:
#  type: NodePort
#  selector:
#    name_service_cr: name-service
#  ports:
#    - port: 9876
#      targetPort: 9876
#      # use this port to access the name server cluster
#      nodePort: 30001
#---
```

#### 部署集群

```shell
# 安装rocketmq集群
[root@k8s-master01 rocketmq-operator]# kubectl apply -f example/rocketmq_v1alpha1_rocketmq_cluster.yaml

# 安装rocketmq集群service
[root@k8s-master01 rocketmq-operator]# kubectl apply -f example/rocketmq_v1alpha1_cluster_service.yaml
```

#### 验证集群

```shell
# 查看rocketmq集群pod
[root@k8s-master01 rocketmq-operator]# kubectl get pod -n test
NAME                               READY   STATUS    RESTARTS   AGE
broker-0-master-0                  1/1     Running   0          13m
broker-0-replica-1-0               1/1     Running   0          13m
console-fd66cc958-t7twh            1/1     Running   0          13m
name-service-0                     1/1     Running   0          13m
rocketmq-operator-867c4955-dhgzh   1/1     Running   0          13m

# 查看rocketmq集群svc
[root@k8s-master01 rocketmq-operator]# kubectl get svc -n test
NAME                                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
console-service                                          NodePort    10.233.46.31    <none>        8080:30849/TCP   12m
glusterfs-dynamic-0fc569c2-e2fe-4ef1-be6a-b3d56f1058d1   ClusterIP   10.233.36.32    <none>        1/TCP            15h
glusterfs-dynamic-ceed9eef-7264-45c5-b727-4afc64ab34ab   ClusterIP   10.233.60.200   <none>        1/TCP            16h
glusterfs-dynamic-e7c85113-e164-4423-a5ad-99f7c3f8b0f1   ClusterIP   10.233.24.21    <none>        1/TCP            15h
rocketmq-operator                                        ClusterIP   10.233.61.255   <none>        8383/TCP         13m

# 查看rocketmq集群pvc
[root@k8s-master01 rocketmq-operator]# kubectl get pvc -n test
NAME                                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
broker-storage-broker-0-master-0      Bound    pvc-e7c85113-e164-4423-a5ad-99f7c3f8b0f1   8Gi        RWO            glusterfs      15h
broker-storage-broker-0-replica-1-0   Bound    pvc-0fc569c2-e2fe-4ef1-be6a-b3d56f1058d1   8Gi        RWO            glusterfs      15h
namesrv-storage-name-service-0        Bound    pvc-ceed9eef-7264-45c5-b727-4afc64ab34ab   10Gi       RWO            glusterfs      16h

# 浏览器访问nodeip+console-service的nodeport端口进行验证 
# http://ip:30849/
```



## k8s部署XXL-JOB

### 部署步骤

#### 创建XXL-JOB所需数据库

```shell
# 数据库sql参考地址包含创建数据库步骤,见附录
[root@k8s-master-1 ~]# wget https://github.com/xuxueli/xxl-job/blob/master/doc/db/tables_xxl_job.sql
#查看mysql的连接端口
[root@k8s-master-1 ~]# kubectl get svc -n test
NAME                                                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
mysql                                                    NodePort    10.233.20.110   <none>        3306:30850/TCP   47m

# 使用mysql连接工具连接数据库并导入上方sql，因为mysql service的类型为nodeport,所以连接地址为任一k8s集群的节点ip地址，端口为30850
# 以下创建用户操作也可以再连接工具中操作
# 登录数据库
[root@k8s-master-1 ~]# kubectl exec -it mysql-589dcf6597-5ps6x /bin/bash -n test
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mysql-589dcf6597-5ps6x:/# mysql -u root -p
Enter password:

mysql> create user 'xxl'@'%' identified by 'P@ssw0rd@xxl';                #创建xxl用户用于xxl-job服务连接
Query OK, 0 rows affected (0.00 sec)

mysql> GRANT ALL PRIVILEGES ON  xxl_job.* TO 'xxl'@'%' IDENTIFIED BY 'P@ssw0rd@xxl';     #对xxl用户授权
Query OK, 0 rows affected (0.00 sec)

mysql> FLUSH PRIVILEGES;                                    #刷新权限
Query OK, 0 rows affected (0.00 sec)
```

#### 创建XXL-JOB的副本yaml文件

```shell
[root@k8s-master-0 ~]# vi xxl-job.yaml
```

- xxl-job.yaml

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: xxl-job
  namespace: test
  labels:
    app: xxl-job
  annotations:
    deployment.kubernetes.io/revision: '6'
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xxl-job
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: xxl-job
      annotations:
        cni.projectcalico.org/ipv4pools: '["default-ipv4-ippool"]'
    spec:
      containers:
        - name: container-e0tn05
          image: 'xuxueli/xxl-job-admin:2.3.0'
          ports:
            - name: tcp-8080
              containerPort: 8080
              protocol: TCP
          env:
            - name: PARAMS
              value: >-
                --spring.datasource.url=jdbc:mysql://mysql.test:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
                --spring.datasource.username=xxl
                --spring.datasource.password=P@ssw0rd@xxl
                --xxl.job.accessToken=RfCwgzKLuRGbrqqN9Tg9WT3t                #注意修改数据库连接地址端口及用户密码
          resources:
            limits:
              cpu: '4'
              memory: 8000Mi
            requests:
              cpu: 500m
              memory: 500Mi
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          imagePullPolicy: IfNotPresent
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      dnsPolicy: ClusterFirst
      serviceAccountName: default
      serviceAccount: default
      securityContext: {}
      affinity: {}
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```

#### 创建XXL-JOB的service yaml文件

```shell
[root@k8s-master-0 ~]# vi xxl-job-service.yaml
```

- xxl-job-service.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  name: test-xxl-job-service
  namespace: test
  labels:
    app: test-xxl-job-service
spec:
  ports:
    - name: http-8080
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30850
  selector:
    app: test-xxl-job
  type: NodePort
  sessionAffinity: None
  externalTrafficPolicy: Cluster
```

#### 创建XXL-JOB

```shell
# 部署xxl_job副本
[root@k8s-master-1 ~]# kubectl apply -f  xxl-job.yaml

# 部署xxl_job服务svc
[root@k8s-master-1 ~]# kubectl apply -f  xxl-job-service.yaml
```

#### 验证XXL-JOB

```shell
# 查看xxl-job的pod
[root@k8s-master-1 ~]# kubectl get pod -n test

# 查看xxl-job的svc
[root@k8s-master-1 ~]# kubectl get svc -n test

# 浏览器访问任一节点ip+30850（nodeport端口）进行验证 
# http://ip:30850/xxl-job-admin/
# 默认用户名/密码 ：admin/123456
```



## k8s部署SkyWalking

### 部署步骤

#### 创建SkyWalking的配置 yaml文件

```shell
[root@k8s-master-1 ~]# vi skywalking-cm.yaml
```

- skywalking-cm.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skywalking-cm
  namespace: test
data:
  STORAGE: 'elasticsearch7'
  STORAGE_ES_CLUSTER_NODES: '*.*.*.*:9200'                #es的连接地址
  ES_USER: '****'                                            #es的用户名
  ES_PASSWORD: '********'                                    #es的密码
  CORE_GRPC_PORT: '11800'
  CORE_REST_PORT: '12800'
```

#### 创建SkyWalking的副本 yaml文件

```shell
[root@k8s-master-1 ~]# vi skywalking-deployment.yaml
```

- skywalking-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: skywalking
  name: test-skywalk-skywalking-oap
  namespace: test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: skywalking
  template:
    metadata:
      labels:
        app: skywalking
    spec:
      containers:
        - envFrom:
          - prefix: SW_
            configMapRef:
              name: skywalking-cm
          image: apache/skywalking-oap-server:8.7.0-es7
          imagePullPolicy: IfNotPresent
          name: skywalking
          ports:
            - containerPort: 12800
              name: http
              protocol: TCP
            - containerPort: 11800
              name: grpc
              protocol: TCP
          resources:
            limits:
              cpu: '2'
              memory: 2Gi
            requests:
              cpu: '1'
              memory: 2Gi
          volumeMounts:
            - mountPath: /etc/localtime
              name: volume-localtime
      volumes:
        - hostPath:
            path: /etc/localtime
            type: ''
          name: volume-localtime
```

#### 创建SkyWalking的service yaml文件

```shell
[root@k8s-master-1 ~]# vi skywalking-deployment.yaml
```

- skywalking-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-skywalk-skywalking-oap
  namespace: test
  labels:
    app: skywalking
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 12800
      protocol: TCP
      targetPort: 12800
    - name: grpc
      port: 11800
      protocol: TCP
      targetPort: 11800
  selector:
    app: skywalking
```

#### 创建SkyWalking-ui副本的service yaml文件

```shell
[root@k8s-master-1 ~]# vi skywalking-ui-service.yaml
```

- skywalking-ui-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: skywalking-ui
  labels:
    app: skywalking-ui
  namespace: test
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: skywalking-ui
```

#### 创建SkyWalking-ui的副本yaml文件

```shell
[root@k8s-master-1 ~]# vi skywalking-ui.yaml
```

- skywalking-ui.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: skywalking-ui
  name: skywalking-ui
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: skywalking-ui
  template:
    metadata:
      labels:
        app: skywalking-ui
    spec:
      containers:
        - env:
            - name: SW_OAP_ADDRESS
              value: "skywalking:12800"
          image: apache/skywalking-ui:8.9.1
          imagePullPolicy: IfNotPresent
          name: skywalking-ui
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          resources:
            limits:
              cpu: '2'
              memory: 1Gi
            requests:
              cpu: '1'
              memory: 1Gi
          volumeMounts:
            - mountPath: /etc/localtime
              name: volume-localtime
      volumes:
        - hostPath:
            path: /etc/localtime
            type: ''
          name: volume-localtime
```

#### 部署安装SkyWalking

```shell
[root@k8s-master01 ~]# kubectl apply -f skywalking-cm.yaml
[root@k8s-master01 ~]# kubectl apply -f skywalking-deployment.yaml
[root@k8s-master01 ~]# kubectl apply -f skywalking-service.yaml
[root@k8s-master01 ~]# kubectl apply -f skywalking-ui-service.yaml
[root@k8s-master01 ~]# kubectl apply -f skywalking-ui.yaml
```

#### 验证SkyWalking

```shell
# 查看SkyWalking副本
[root@k8s-master01 ~]# kubectl get pod -n test
NAME                                                  READY   STATUS    RESTARTS   AGE
skywalking-ui-7cb7f68686-4sgtq                        1/1     Running   0          22d
test-skywalk-skywalking-oap-67f6cd45fd-dm7d4          1/1     Running   0          22d
test-skywalk-skywalking-oap-67f6cd45fd-ggg4z          1/1     Running   0          22d

# 查看SkyWalking service
[root@k8s-master01 ~]# kubectl get svc -n test 
NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
skywalking-ui                           NodePort    10.233.16.121   <none>        8080:30849/TCP                        22d
test-skywalk-skywalking-oap             ClusterIP   10.233.12.1     <none>        12800/TCP,11800/TCP                   22d

# 使用浏览器访问任意节点ip+30849（nodeport端口）进行验证
```



> **参考链接**

1. [Nacos]([nacos-k8s/README-CN.md at master · nacos-group/nacos-k8s · GitHub](https://github.com/nacos-group/nacos-k8s/blob/master/README-CN.md))
2. [RocketMQ](https://github.com/apache/rocketmq-operator)
3. [XXL-JOB](https://www.xuxueli.com/xxl-job/)
4. [SkyWalking](https://skywalking.apache.org/)

> **后续**

1. 基于KubeSphere的Kubernetes生产实践之路-SpingCloud业务模块的devops自动化部署
