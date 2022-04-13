## 在EKS中使用fluent-bit收集各pod的日志(kubectl logs)

本文将介绍如何使用fluent-bit收集kubectl logs podname所看到的log。 本质上， kubectl logs podname命令所看到的日志信息是存在对应的eks node的/var/log/containers目录下. Pod中所运行的程序所有的stdout输出都可以用kubectl logs来查看。


### 创建S3 bucket
可采用下面AWS Cli命令进行创建S3 bucket
aws s3 mb s3://fluent-bit-s3-test --region cn-north-1

### 创建iamserviceaccount
FluentBit在将日志上传到S3时，需要S3的 PutObject 权限。

需要执行下面指令，创建 iamserviceaccount，进行授权。

```
    cat <<EOF > s3-putobject-iam-policy.json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:PutObject"
                ],
                "Resource": "arn:aws-cn:s3:::fluent-bit-s3-test/*"
            }
        ]
    }
    EOF
    
    aws iam create-policy --policy-name S3PutObjectPolicy --policy-document file://./s3-putobject-iam-policy.json --region cn-north-1
    
    POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==` S3PutObjectPolicy`].Arn' --output text --region cn-north-1)
    
    eksctl utils associate-iam-oidc-provider --cluster=eks-bjs --approve --region cn-north-1
    
    eksctl create iamserviceaccount \
        --cluster=eks-bjs \
        --region cn-north-1 \
        --namespace=default \
        --name=s3-putobject \
        --attach-policy-arn=${POLICY_NAME} \
        --override-existing-serviceaccounts \
        --approve

```


如果已有符合条件的policy， 则可以直接使用现有的policy
下图替换s3-putobject-iam-policy.json里的Resource / 前面的部分
![查看状态](/images/2.jpg)

下图替换eksctl create iamserviceaccount里--attach-policy-arn参数
![查看状态](/images/1.png)

注意如果使用已有的policy， 那么只需要执行eksctl utils associate-iam-oidc-provider和eksctl create iamserviceaccount这2个命令


### 创建测试用的testapp
在测试 testapp 中，运行一个简单shell脚本，这个脚本将日志打印到stdout
测试 testapp 的 yml 文件如下：

```
cat <<'EOF' > testapp.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  labels:
    app: test
spec:
  selector:
    matchLabels:
      app: test
  replicas: 4
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: createlog
        image: busybox
        imagePullPolicy: Always
        command: [ "sh", "-c"]
        args:
        - i=0;
          while true; 
            do
              echo "[`date +%Y/%m/%d-%H:%M:%S`]"+$i+$HOSTNAME;
              let i=i+1; 
              sleep 2;
            done
EOF
```
安装testapp的命令如下：
kubectl apply -f testapp.yml

其中createlog容器的输出日志格式如下, 查看方式为kubectl logs podname --namespace default：

[2020/11/12-02:58:37]+103+test-deployment-6db76c8fff-j6hhs


### 创建fluent bit config
对于EKS， pod里面运行的程序输出到stdout的内容会保存到运行该pod的node的/var/log/containers目录下

fluent bit config的示例如下：

其中[INPUT] Path /var/log/containers/*.log为各个pod的标准输出日志的保存位置 fluent bit将该目录中*.log文件中的日志内容保存到S3中。

[OUTPUT]中的total_file_size参数是设定S3日志文件的最大尺寸，upload_timeout 参数是设定每次上传日志文件的最大间隔时间。详细参数说明见：https://docs.fluentbit.io/manual/pipeline/outputs/s3

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentbit-config
  labels:
    k8s-app: fluentbit
data:
# Configuration files: server, input, filters and output
# ======================================================
  fluentbit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off

    [INPUT]
        Name              tail
        Tag               test-log
        Path              /var/log/containers/xxx*.log
        DB                /var/log/testlog.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   Off
        Refresh_Interval  10

    [OUTPUT]
        Name                          s3
        Match                         test-log
        bucket                        fluent-bit-s3-test
        region                        cn-northwest-1
        store_dir                     /home
        total_file_size               10M
        upload_timeout                1m
        s3_key_format                 /fluent-bit-test/log/year=%Y/month=%m/day=%d/%H-%M-%S
```

安装 fluent bit config 的命令如下：

kubectl apply -f fluentbit-config-s3.yml

### 创建fluent bit daemonset
fluent bit daemonset的示例如下：

其中需要配置serviceAccountName: s3-putobject


```
cat <<EOF > fluentbit-daemonset.yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
    name: fluentbit
    labels:
        k8s-app: fluentbit
        version: v8
        kubernetes.io/cluster-service: "true"
spec:
    selector:
        matchLabels:
          k8s-app: fluentbit
          version: v1
    updateStrategy:
        type: RollingUpdate
    template:
        metadata:
            labels:
                k8s-app: fluentbit
                version: v1
                kubernetes.io/cluster-service: "true"
        spec:
            containers:
              - name: fluentbit
                image: fluent/fluent-bit:1.6.0
                imagePullPolicy: Always
                command: ["/fluent-bit/bin/fluent-bit","-c", "/fluent-bit/etc/fluentbit.conf"]
                env:
                - name: NODE_NAME
                  valueFrom:
                    fieldRef:
                        fieldPath: spec.nodeName
                - name: MY_POD_NAME
                  valueFrom:
                    fieldRef:
                        fieldPath: metadata.name
                - name: MY_POD_NAMESPACE
                  valueFrom:
                    fieldRef:
                        fieldPath: metadata.namespace
                - name: MY_POD_IP
                  valueFrom:
                    fieldRef:
                        fieldPath: status.podIP
                resources:
                    requests:
                        cpu: 5m
                        memory: 20Mi
                    limits:
                        cpu: 60m
                        memory: 60Mi
                volumeMounts:
                - name: varlog
                  mountPath: /var/log
                - name: var-lib-docker-containers
                  mountPath: /var/lib/docker/containers
                  readOnly: true
                - name: fluentbit-config
                  mountPath: /fluent-bit/etc/
            serviceAccountName: s3-putobject
            terminationGracePeriodSeconds: 10
            volumes:
                - name: varlog
                  hostPath:
                    path: /var/log
                - name: var-lib-docker-containers
                  hostPath:
                    path: /var/lib/docker/containers
                - name: fluentbit-config
                  configMap:
                    name: fluentbit-config
EOF

```

需要注意的是， fluent-bit不能处理软链文件， 因此需要把/var/lib/docker/containers目录也挂载到daemonset里

<b>
/var/log/containers/*.log are symlinks to /var/log/pods/* which are also symlinks to /var/lib/docker/containers/*
You must mount them all to your fluentbit pods for proper work:
</b>

安装fluent bit daemonset的命令如下：

kubectl apply -f fluentbit-daemonset.yml


### 查看fluent bit日志，查看在s3中保存的日志
查看fluent bit pod是否运行正常，命令如下：

kubectl get pod

下面是命令的返回结果，可以看到fluentbit-*是Running状态。

![查看状态](/images/3.jpg)

查看fluent bit pod的日志，命令如下：

kubectl logs -l k8s-app=fluentbit


[参考文档](https://aws.amazon.com/cn/blogs/china/scheme-of-using-fluent-bit-in-eks-to-collect-application-logs-and-save-them-in-s3/)