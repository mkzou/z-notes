# KubeSphere DevOps 系统

大家好，我是老Z。

本文接着上篇 **<<基于KubeSphere的Kubernetes生产实践之路-SpingCloud架构中间件的安装部署>>** 继续打造我们Kubernetes生产环境。

## 前提说明

1. 基于 Jenkins 的 KubeSphere DevOps 系统是专为 Kubernetes 中的 CI/CD 工作流设计的，它提供了一站式的解决方案，帮助开发和运维团队用非常简单的方式构建、测试和发布应用到 Kubernetes。它还具有插件管理、Binary-to-Image (B2I)、Source-to-Image (S2I)、代码依赖缓存、代码质量分析、流水线日志等功能。

2. KubeSphere DevOps 系统为用户提供了一个自动化的环境，应用可以自动发布到同一个平台。它还兼容第三方私有镜像仓库（如 Harbor）和代码库（如 GitLab/GitHub/SVN/BitBucket）。它为用户提供了全面的、可视化的 CI/CD 流水线，打造了极佳的用户体验，而且这种兼容性强的流水线能力在离线环境中非常有用，本文就是在离线环境实现了CI/CD 流水线。

3. 需要提前安装配置Harbor和GitLab(本文略过)。

4. DevOps流程架构图
   
   ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-flow.png)

## KubeSphere开启DevOps系统

### 在部署Kubernetes之前

1. 编辑集群部署配置文件

```shell
[root@k8s-master-0 ~]# vim config-sample.yaml
```

2. 在该文件中，搜索 `devops`，并将 `enabled` 的 `false `改为 `true`，完成后保存文件

```yaml
  devops:
    enabled: true     # 将“false”更改为“true”
    jenkinsMemoryLim: 2Gi
    jenkinsMemoryReq: 1500Mi
    jenkinsVolumeSize: 8Gi
    jenkinsJavaOpts_Xms: 512m
    jenkinsJavaOpts_Xmx: 512m
    jenkinsJavaOpts_MaxRAM: 2g
```

3. 使用集群部署配置文件创建集群

```shell
[root@k8s-master-0 ~]# ./kk create cluster -f config-sample.yaml
```

### 在部署Kubernetes之后

1. 以 `admin` 用户KubeSphere登录控制台，点击左上角的**平台管理**，选择**集群管理**。
   
   ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-28.jpg)

2. 点击 **自定义资源CRD**，在搜索栏中输入 `clusterconfiguration`，点击搜索结果查看其详细页面
   
   - 自定义资源（CRD）允许用户在不新增 API 服务器的情况下创建一种新的资源类型，用户可以像使用其他 Kubernetes 原生对象一样使用这些定制资源。
   
   ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-28.jpg)

3. 在**自定义资源**中，点击 `ks-installer` ，选择**编辑 YAML**。![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-29.jpg) 
   
   - 在该 YAML 文件中，搜索 `devops`，将 `enabled` 的 `false` 改为 `true`。完成后，点击右下角的**更新**，保存配置，保存之后KubeSphere 会**自动安装**devops插件。
   
   ```yaml
   devops:
     enabled: true  # 将“false”更改为“true”
   ```
   
   ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-30.jpg)

### 验证

- 在 kubectl 中执行以下命令检查安装过程

```shell
[root@k8s-master-0 ~]# kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

- 检查Kubernetes集群devops命名空间的pod运行状态

```shell
[root@k8s-master-0 ~]# kubectl get pod -n kubesphere-devops-system
NAME                          READY   STATUS    RESTARTS   AGE
ks-jenkins-677cdbc9d8-wxzxp   1/1     Running   0          19m
s2ioperator-0                 1/1     Running   0          19m
```

## 使用DevOps系统

### 准备工作

1. 创建企业空间
- 使用admin账户登录KubeSphere平台，点击工作台，进入企业空间![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-1.jpg)

- 创建名称为test的企业空间![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-0.jpg)

- 进入新建的test企业空间，点击企业空间设置，可以对企业空间进行一系列配置
  
  - 基本信息：修改企业空间的基本信息和删除企业空间
  
  - 配额管理：对此企业空间进行资源配额
  
  - 企业角色：新增自定义权限的企业角色
  
  - 企业成员：新增和删除成员，修改成员权限
  
  - 企业组织：对成员进行组织分类![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-2.jpg)
2. 为企业空间创建项目
- 点击项目管理，创建新项目（KubeSphere 中的项目对应的是 Kubernetes 的 namespace）![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-3.jpg)

- 查看创建好的项目，点击右方三个点可以对项目进行基本信息的修改和项目资源的配额等![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-4.jpg)

- 点击项目可以查看和配置此项目的一系列资源![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-5.jpg)
3. 创建DevOps 工程与凭证
- 点击DevOps 工程进行新建名为test的DevOps 工程![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-6.jpg)

- 进入test DevOps 工程，点击凭证，创建以下凭证
  
  | 凭证 ID           | 类型         | 用途              |
  |:--------------- |:---------- |:--------------- |
  | dockerhub-id    | 帐户凭证       | 镜像仓库的拉取凭证       |
  | github-id       | 帐户凭证       | Git仓库的拉取凭证      |
  | demo-kubeconfig | kubeconfig | Kubernetes的操作凭证 |
  
  - dockerhub-id![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-8.jpg)
  
  - github-id![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-9.jpg)
  
  - demo-kubeconfig
    
    创建此凭证时，类型选择kubeconfig，Content会自动生成![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-10.jpg)

### 配置DevOps流水线

#### 使用Jenkinsfile创建流水线

- 点击流水线，新建名为devops的流水线![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-11.jpg)

- 点击下一步，进行流水线的基础配置，完成创建![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-12.jpg)
2. 配置Jenkinsfile
- 点击流水线，点击编辑Jenkinsfile![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-13.jpg)

- 因前后端编译方式的差异，这里将分别举例前后端Jenkinsfile配置
  
  - 前端
  
  ```Jenkinsfile
  pipeline {
    agent {
      node {
        label 'nodejs'
      }
    }
  
    parameters {
      // 这里定义的参数关键字、默认值、注释都会在运行流水线时显示出来
      string(name:'APP_BRANCH', defaultValue: 'test', description: '构建应用的代码分支或tag名称')
      string(name:'IMAGE_TAG', defaultValue: '0.1.1', description: '构建应用的镜像tag名称')
      string(name:'REPLICA', defaultValue:  '1', description: '构建应用的副本数量')
    }
  
    environment {
      // Harbor 仓库的地址
      DOCKERHUB_REGISTRY = 'mydockerhub.com'
  
      // K8s上创建的Harbor仓库凭证ID
      DOCKERHUB_CREDENTIAL_ID = 'dockerhub-id'
  
      // Harbor 项目仓库命名空间
      DOCKERHUB_NAMESPACE = 'test'
      DOCKERHUB_REPOSITORY = "$DOCKERHUB_REGISTRY/$DOCKERHUB_NAMESPACE"
  
      // GitLab 仓库的地址
      GITLAB = 'mygitlab.com'
  
      // K8s上创建的GitLab仓库凭证ID
      GITLAB_CREDENTIAL_ID = 'github-id'
  
      // GitLab 项目仓库命名空间
      GITLAB_NAMESPACE = 'test'
  
      // 构建应用的项目名称
      APP_NAME = 'test-web'
  
      // 构建应用的devops项目名称
      APP_DEVOPS_NAME = "$APP_NAME-devops"
  
      // 构建的临时目录
      BUILD_TMP = '/tmp'
  
      // K8s上创建的kubeconfig凭证ID
      KUBECONFIG_CREDENTIAL_ID = 'demo-kubeconfig'
  
    }
  
    stages {
    // 拉取GitLab 代码
     stage('checkout scm') {
        steps{
          container ('nodejs') {
            withCredentials([usernamePassword(credentialsId : "$GITLAB_CREDENTIAL_ID" ,passwordVariable : 'GITLAB_PASSWORD' ,usernameVariable : 'GITLAB_USERNAME' ,)]) {
              sh 'cd $BUILD_TMP && git clone -b $APP_BRANCH https://$GITLAB_USERNAME:$GITLAB_PASSWORD@$GITLAB/$GITLAB_NAMESPACE/$APP_NAME.git'
            }
          }
        }
      }
     // npm 编译
      stage('node build') {
        steps {
          container ('nodejs') {
            sh 'cd $BUILD_TMP/$APP_NAME && npm install && npm run build'
          }
        }
      }
      // 登录镜像仓库 根据GitLab中的Dockerfile将代码打成镜像 并推送到镜像仓库
      stage('docker build && push') {
        steps {
          container ('nodejs') {
            withCredentials([usernamePassword(credentialsId : "$DOCKERHUB_CREDENTIAL_ID" ,passwordVariable : 'DOCKERHUB_PASSWORD' ,usernameVariable : 'DOCKERHUB_USERNAME' ,)]) {
              sh 'echo "$DOCKERHUB_PASSWORD" | docker login $DOCKERHUB_REGISTRY -u "$DOCKERHUB_USERNAME" --password-stdin'
              sh 'cd $BUILD_TMP/$APP_NAME && docker build --no-cache -t $DOCKERHUB_REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$IMAGE_TAG .'
              sh 'docker push $DOCKERHUB_REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$IMAGE_TAG'
           } 
          }
        }
      }
      // 拉取部署模块的yaml模板文件的GitLab代码
      stage('checkout deploy scm') {
        steps {
          withCredentials([usernamePassword(credentialsId : "$GITLAB_CREDENTIAL_ID" ,passwordVariable : 'GITLAB_PASSWORD' ,usernameVariable : 'GITLAB_USERNAME' ,)]) {
            sh 'git clone https://$GITLAB_USERNAME:$GITLAB_PASSWORD@$GITLAB/$GITLAB_NAMESPACE/$APP_DEVOPS_NAME.git'
          }
  
        }
      }
      // 修改yaml模块文件的默认配置(镜像tag、副本数量)
      stage('deploy to k8s') {
        steps {
          sh 'sed -i "s/0.0.1/$IMAGE_TAG/g" $APP_DEVOPS_NAME/deploy/*.yaml'
          sh 'sed -i "s/REPLICA/$REPLICA/g" $APP_DEVOPS_NAME/deploy/*.yaml'
          kubernetesDeploy(configs: "$APP_DEVOPS_NAME/deploy/test-web.yaml", enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
        }
      }
  
    } 
  }
  ```
  
  - 后端
  
  ```Jenkinsfile
  pipeline {
    agent {
      node {
        label 'maven'
      }
    }
  
    parameters {
      // 这里定义的参数关键字、默认值、注释都会在运行流水线时显示出来
      string(name:'APP_BRANCH', defaultValue: 'test', description: '构建应用的代码分支或tag名称')
      string(name:'IMAGE_TAG', defaultValue: '0.1.1', description: '构建应用的镜像tag名称')
      string(name:'REPLICA', defaultValue:  '1', description: '构建应用的副本数量')
      string(name:'ENVIRONMENT', defaultValue: 'test', description: '读取nacos的分组')
    }
  
    environment {
      // Harbor 仓库的地址
      DOCKERHUB_REGISTRY = 'mydockerhub.com'
  
      // K8s上创建的Harbor仓库凭证ID
      DOCKERHUB_CREDENTIAL_ID = 'dockerhub-id'
  
      // Harbor 项目仓库命名空间
      DOCKERHUB_NAMESPACE = 'test'
      DOCKERHUB_REPOSITORY = "$DOCKERHUB_REGISTRY/$DOCKERHUB_NAMESPACE"
  
      // GitLab 仓库的地址
      GITLAB = 'mygitlab.com'
  
      // K8s上创建的GitLab仓库凭证ID
      GITLAB_CREDENTIAL_ID = 'github-id'
  
      // GitLab 项目仓库命名空间
      GITLAB_NAMESPACE = 'test'
  
      // 构建应用的项目名称
      APP_NAME = 'test-service'
  
      // 构建应用的devops项目名称
      APP_DEVOPS_NAME = "$APP_NAME-devops"
  
      // 构建的临时目录
      BUILD_TMP = '/tmp'
  
      // K8s上创建的kubeconfig凭证ID
      KUBECONFIG_CREDENTIAL_ID = 'demo-kubeconfig'
  
    }
  
    stages {
      // 拉取GitLab 代码
      stage('checkout scm') {
        steps{
          container ('maven') {
            withCredentials([usernamePassword(credentialsId : "$GITLAB_CREDENTIAL_ID" ,passwordVariable : 'GITLAB_PASSWORD' ,usernameVariable : 'GITLAB_USERNAME' ,)]) {
              sh 'cd $BUILD_TMP && git clone -b $APP_BRANCH https://$GITLAB_USERNAME:$GITLAB_PASSWORD@$GITLAB/$GITLAB_NAMESPACE/$APP_NAME.git'
            }
          }
        }
      }
      // maven 编译
      stage('maven build') {
        steps {
          container ('maven') {
            sh 'cd $BUILD_TMP/$APP_NAME && mvn clean package -Dmaven.test.skip=true'
          }
        }
      }
      // 登录镜像仓库 根据GitLab中的Dockerfile将代码打成镜像 并推送到镜像仓库
      stage('docker build && push') {
        steps {
          container ('maven') {
            withCredentials([usernamePassword(credentialsId : "$DOCKERHUB_CREDENTIAL_ID" ,passwordVariable : 'DOCKERHUB_PASSWORD' ,usernameVariable : 'DOCKERHUB_USERNAME' ,)]) {
             sh 'cd $BUILD_TMP/$APP_NAME && mvn -f test-service dockerfile:build dockerfile:push -Ddocker.repository=$DOCKERHUB_REPOSITORY -Ddockerfile.username=$DOCKERHUB_USERNAME -Ddockerfile.password=$DOCKERHUB_PASSWORD'
            }
          }
        }
      }
      // 拉取部署模块的yaml模板文件的GitLab代码
      stage('checkout deploy scm') {
        steps {
          withCredentials([usernamePassword(credentialsId : "$GITLAB_CREDENTIAL_ID" ,passwordVariable : 'GITLAB_PASSWORD' ,usernameVariable : 'GITLAB_USERNAME' ,)]) {
            sh 'git clone -b test https://$GITLAB_USERNAME:$GITLAB_PASSWORD@$GITLAB/$GITLAB_NAMESPACE/$APP_DEVOPS_NAME.git'
          }
  
        }
      }
      // 修改yaml模块文件的默认配置(镜像tag、副本数量、nacos命名空间)
      stage('deploy to dev') {
        steps {
          sh 'sed -i "s/0.0.1/$IMAGE_TAG/g" $APP_DEVOPS_NAME/deploy/*.yaml'
          sh 'sed -i "s/REPLICA/$REPLICA/g" $APP_DEVOPS_NAME/deploy/*.yaml'
          sh 'sed -i "s/DEFAULT_GROUP/$ENVIRONMENT/g" $APP_DEVOPS_NAME/deploy/*.yaml'
          kubernetesDeploy(configs: "$APP_DEVOPS_NAME/deploy/test-service", enableConfigSubstitution: true, kubeconfigId: "$KUBECONFIG_CREDENTIAL_ID")
        }
      }
    }
  }
  ```
3. 验证
- 创建完毕后，点击运行即可看到我们在Jenkinsfile中定义的参数![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-14.jpg)

#### 创建镜像仓库拉取凭证

**在KubeSphere平台创建镜像仓库的拉取凭证harbor-secret，用于创建模块时自动拉取镜像**

- 创建名为harbor-secret的secret凭证，进入平台管理，点击配置中心，选择密钥，创建凭证![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-18.jpg)

- 类型选择kubernetes.io/dockerconfigjson (镜像仓库密钥)，并配置密钥信息，点击验证进行测试，验证通过即可保存![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-19.jpg)

#### 配置部署模块的yaml文件

**在Git仓库中创建存储部署模块的yaml模板文件**

- 前端
  
  ```yaml
  kind: Deployment
  apiVersion: apps/v1
  metadata:
    name: test-web
    namespace: test
    labels:
      app: test-web
  spec:
    replicas: REPLICA                                    #定义副本数量，在devops部署时会根据填写的变量进行替换
    selector:
      matchLabels:
        app: test-web
    template:
      metadata:
        labels:
          app: test-web
      spec:
        containers:
          - name: test-web
            image: 'mydockerhub.com/test/test-web:0.0.1'        #定义镜像名称，镜像版本号会根据devops部署时填写的变量进行替换
            ports:
              - name: http-80
                containerPort: 80                    #定义前端程序内部nginx启动的端口
                protocol: TCP
            resources:                            #资源配额
              limits:
                cpu: '1'
                memory: 4Gi
              requests:
                cpu: 50m
                memory: 500Mi
            volumeMounts:
              - name: host-time
                readOnly: true
                mountPath: /etc/localtime
              - name: nginx-conf                    #挂载nginx转发配置文件
                readOnly: true
                mountPath: /etc/nginx/conf.d/
            imagePullPolicy: Always
        volumes:
          - name: host-time
            hostPath:
              path: /etc/localtime
              type: ''
          - name: nginx-conf
            configMap:
              name: nginx-conf
              items:
                - key: default.conf
                  path: default.conf
              defaultMode: 420
        imagePullSecrets:                            #镜像拉取凭证
          - name: harbor-secret
  ---
  kind: Service
  apiVersion: v1
  metadata:
    name: test-web-service
    namespace: test
    labels:
      app: test-web-service
  spec:
    ports:
      - name: http-service
        protocol: TCP
        port: 80
        targetPort: 80
    selector:
      app: test-web
    type: ClusterIP
  ```

- 后端
  
  ```yaml
  kind: Deployment
  apiVersion: apps/v1
  metadata:
    name: test-service
    namespace: test
    labels:
      app: test-service
  spec:
    replicas: REPLICA                                        #定义副本数量，在devops部署时会根据填写的变量进行替换
    selector:
      matchLabels:
        app: test-service
    template:
      metadata:
        labels:
          app: test-service
      spec:
        volumes:
          - name: volume-log                                #定义程序的日志持久化pvc
            persistentVolumeClaim:
              claimName: test-service-log
        containers:
          - name: test-service
            image: 'mydockerhub.com/test/test-service:0.0.1'        #定义镜像名称，镜像版本号会根据devops部署时填写的变量进行替换
            ports:
              - name: http-server
                containerPort: 9000                #程序端口
                protocol: TCP
            env:                                            ##下列环境变量均为示例，实际情况按照程序的具体配置配置
              - name: APP_PORT                            #后端程序的端口
                value: '9000'
              - name: APP_DEPLOY                            #定义连接nacos的哪个命名空间，会根据devops部署时填写的变量进行替换
                value: 'DEFAULT_GROUP'
              - name: NACOS_SERVER_IP                        #定义nacos的链接地址
                value: 'nacos-server.test.svc.cluster.local'
              - name: NACOS_SERVER_PORT                    #定义nacos的连接端口
                value: '8848'
              - name: SW_AGENT_NAMESPACE                    #skywalking的agent的命名空间
                value: test
              - name: SW_AGENT_NAME                        #skywalking的agent的名称
                value: test-service
              - name: SW_AGENT_COLLECTOR_BACKEND_SERVICES                            #定义skywalking的连接地址
                value: test-skywalk-skywalking-oap.test.svc.cluster.local:11800
              - name: SW_GRPC_LOG_SERVER_HOST
                value: test-skywalk-skywalking-oap.test.svc.cluster.local
              - name: SW_GRPC_LOG_SERVER_PORT
                value: '11800'
              - name: JAVA_OPTS                                                    #skywalking的agent的路径
                value: '-javaagent:/opt/agent/skywalking-agent.jar'
              - name: APP_LOG_PATH                                                #定义程序的日志路径
                value: /app/logs/test-service/
              - name: APP_LOG_NAME                                                #定义日志的名称
                value: 'test-service-${POD_IP}.log'
              - name: APP_LOG_CONFIG                                                #定义nacos中的log配置文件url
                value: http://${spring.cloud.nacos.discovery.server-addr}/nacos/v1/cs/configs?group=${spring.application.deploy}&tenant=${spring.cloud.nacos.discovery.namespace}&dataId=logback-spring.xml
              - name: POD_IP                                                        #此环境变量用于获取pod的ip地址
                valueFrom:
                  fieldRef:
                    apiVersion: v1
                    fieldPath: status.podIP
            resources:                                            #资源配额
              limits:
                cpu: '2'
                memory: 4Gi
              requests:
                cpu: '50m'
                memory: 512Mi
            volumeMounts:                                            #定义程序日志持久化pvc的挂载路径
              - name: volume-log
                mountPath: /app/logs/test-service    
            livenessProbe:                                        #存活探针配置
              tcpSocket:
                port: 9000                                        #定义活性探针的检测端口
              initialDelaySeconds: 60                                #定义初始化等待时间
              timeoutSeconds: 1                                    #定义检测超时时间
              periodSeconds: 20                                    #定义检测周期时间
              successThreshold: 1                                    #定义成功次数
              failureThreshold: 3                                    #定义失败次数
            readinessProbe:                                        #就绪探针配置
              tcpSocket:
                port: 9000
              initialDelaySeconds: 60
              timeoutSeconds: 1
              periodSeconds: 20
              successThreshold: 1
              failureThreshold: 3
            imagePullPolicy: Always            #镜像拉取策略
        imagePullSecrets:                        #镜像拉取凭证
          - name: harbor-secret
  ```

#### 创建前端模块nginx转发配置文件

1. 进入平台管理，点击配置中心，选择配置，创建nginx-conf![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-20.jpg)

2. 添加数据，键填写nginx配置文件名称default.conf，值填写配置文件内容![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-21.jpg)
   
   - 配置文件示例
   
   ```shell
   server {
       listen       80;
       server_name  localhost;
   
       access_log  /var/log/nginx/default.access.log  main;
   
       location / {
           root   /usr/share/nginx/html;
           index  index.html index.htm;
       }
   
       error_page 404 403 500 502 503 504    /40x.html;
       location = /40x.html {
           root   /usr/share/nginx/html;
       }
   
       location ^~ /test2/ {                            #前端转发示例
           proxy_pass http://test2-web-service.test.svc.cluster.local;
           proxy_redirect off;
           proxy_set_header Host $host;
           proxy_set_header REMOTE-HOST $remote_addr;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   
       location ^~ /test/api/ {                    #后端转发示例
           proxy_pass http://test-service.test.svc.cluster.local:9000/test/api/;
           proxy_redirect off;
           proxy_set_header REMOTE-HOST $remote_addr;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
       }
   ```

3. 将前端模块的service模式修改为nodeport类型，以方便集群外部的nginx代理转发
   
   - 进入test项目，点击应该负载下的服务，找到相应前端模块服务，点击右侧三个点，选择编辑外网访问，配置访问方式为NodePort，将生成的端口配置到集群外部的nginx代理配置文件中即可
   
   ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-25.jpg)

#### 运行流水线

- 进入企业空间，找到已创建好的流水线，点击运行，填入相应参数，点击确定。
  
  - APP_BRANCH：构建应用的代码分支或tag名称
  - IMAGE_TAG：构建应用的镜像tag名称
  - REPLICA：构建应用的副本数量
  - ENVIRONMENT：读取nacos配置的分组
  
  ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-22.jpg)

- 进入活动，点击刚才运行的流水线，可查看流水线详细流程
  
  ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-23.jpg)

- 点击查看日志，可查看流水线的详细日志
  
  ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-24.jpg)

#### 验证

- 进入test企业空间，点击项目管理，进入test项目查看相应的工作负载运行情况
  
  ![](https://znotes-1258881081.cos.ap-beijing.myqcloud.com/k8s-on-kubesphere/devops-26.jpg)

## 参考文档

1. [KubeSphere DevOps](https://kubesphere.io/zh/docs/devops-user-guide/understand-and-manage-devops-projects/overview/)

2. https://github.com/kubesphere/devops-maven-sample



## 后续

1. 基于KubeSphere的Kubernetes生产实践之路-SpingCloud业务模块的日志收集实战(暂定)