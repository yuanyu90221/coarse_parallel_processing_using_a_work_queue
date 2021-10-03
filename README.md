# k8s Task Coarse Parallel Processing Using a Work Queue

## 前言

這個章節將要來實作 [Coarse Parallel Processing Using a Work Queue](https://kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/) 這個練習


## 佈署目標

1 建立一個 Message Queue 的服務： 在這個範例使用的是 RabbitMq 也可以使用其他符合 AMPQ 特性的 QUEUE 

2 建立一個 QUEUE 並且填入要發送的訊息進去, 每一個訊息代表必須要執行的 Job

3 透過 k8s 發佈 Job 來執行從 QUEUE 接收過來的任務： 這個 Job 會開啟多個 Pod, 每個 Pod 會從 QUEUE 拿出一個任務來執行, 不斷重複直到 QUEUE 的任務都執行結束


## 建立一個 Message Queue 的服務

建立 rabbitmq-service.yaml 如下：

```yaml=
apiVersion: v1
kind: Service
metadata:
  labels:
    component: rabbitmq
  name: rabbitmq-service
spec:
  ports:
  - port: 5672
  selector:
    app: taskQueue
    component: rabbitmq
```

建制一個 Service 名稱設定為 rabbitmq-service

設定開啟的 Port 為 5672

建制指令如下：
```shell=
kubectl apply -f rabbitmq-service.yaml
```

建立 rabbitmq-controller.yaml 如下：

```yaml=
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    component: rabbitmq
  name: rabbitmq-controller
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: taskQueue
        component: rabbitmq
    spec:
      containers:
      - image: rabbitmq
        name: rabbitmq
        ports:
        - containerPort: 5672
        resources:
          limits:
            cpu: 100m
```

建立一個 ReplicationController 名稱設定為 rabbitmq-controller

設定 image 使用 rabbitmq

設定 containerPort 為 5672

設定 resources limit 為 cpu: 100m

設定 replicas 為 1

建制指令如下：
```shell=
kubectl apply -f rabbitmq-controller.yaml
```

## 建立一個 QUEUE 並且填入要發送的訊息進去

首先需要再同一個 Cluster 內建立一個暫時的 Pod

並且安裝 ampq 的操作工具

建立指令如下：

```shell
kubectl run -i --tty temp --image ubuntu:18.04
```

在操作的 terminal 輸入以下指令來安裝操作工具

```shell=
apt-get update
apt-get install -y curl ca-certificates amqp-tools python dnsutils
```

![](https://i.imgur.com/XX2rut9.png)


察看 rabbitmq-service 的 IP

```shell=
nslookup rabbitmq-service
```

![](https://i.imgur.com/Vu1OGaE.png)

開始建立測試 QUEUE

```shell=
export BROKER_URL=amqp://guest:guest@rabbitmq-service:5672
## create a queue with name foo
/usr/bin/amqp-declare-queue --url=$BROKER_URL -q foo -d
## publish message to foo
/usr/bin/amqp-publish --url=$BROKER_URL -r foo -p -b Hello
## get the message from foo
/usr/bin/amqp-consume --url=$BROKER_URL -q foo -c 1 cat && echo
```

![](https://i.imgur.com/dZUXXOh.png)

建立 QUEUE 並且輸入訊息
```shell=
/usr/bin/amqp-declare-queue --url=$BROKER_URL -q job1  -d
for f in apple banana cherry date fig grape lemon melon
do
  /usr/bin/amqp-publish --url=$BROKER_URL -r job1 -p -b $f
done
```

## 透過 k8s 發佈 Job 來執行從 QUEUE 接收過來的任務

### 建立 image
建立 Dockerfile 如下：
```yaml=
# Specify BROKER_URL and QUEUE when running
FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y curl ca-certificates amqp-tools python \
       --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*
COPY ./worker.py /worker.py

CMD  /usr/bin/amqp-consume --url=$BROKER_URL -q $QUEUE -c 1 /worker.py
```

建立 worker.py 如下：
```python=
#!/usr/bin/env python

# Just prints standard out and sleeps for 10 seconds.
import sys
import time
print("Processing " + sys.stdin.readlines()[0])
time.sleep(10)
```

修改 worker.py 為可執行權限

```shell=
chmod +x worker.py
```
建立 docker image with name job-wq-1

```shell=
docker build -t job-wq-1 .
```

這邊筆者使用 docker hub 做為 image registry 所以下面指令如下

```shell=
docker tag job-wq-1 yuanyu90221/job-wq-1
docker push yuanyu90221/job-wq-1
```

![](https://i.imgur.com/Wzq8pHN.png)

成功 push 的話 就可以到 dockerhub 找到對應的 [job-wq-1](https://hub.docker.com/repository/docker/yuanyu90221/job-wq-1)

![](https://i.imgur.com/VfnqRRG.png)

### 建立 Job 

建立 job.yaml 如下

```yaml=
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: docker.io/yuanyu90221/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure
```

建立一個 Job 名稱設定為 job-wq-1

completions 設定為 8 代表要執行 8 個 job 才完成

parallelism 設定為 2 代表一次可以啟動 2 個 job

注意的是這邊的 image 設定跟剛剛 push 上去的 image registry url 相關

像筆者是使用自己的 dockerbub 帳號, 所以使用 docker.io/yuanyu90221/job-wq-1

查詢執行結果

```shell=
kubectl describe jobs/job-wq-1
```

![](https://i.imgur.com/5Ujh2sy.png)

檢驗 Pod 內容

```shell=
kubectl get pod
kubectl logs job-wq-1-zz729
```
![](https://i.imgur.com/aNCRwof.png)


## 後話

在此範例中, 我們使用 RabbitMq 來做中間的 Message Brokder

必須要透過 RabbitMq 來傳遞任務

而 k8s 官方有其他的 [Job Pattern](https://kubernetes.io/docs/concepts/workloads/controllers/job/#job-patterns) 可以設定來取代這個作法, 不需要去設定一個 Message Queue Service

而這個範例用一個 Pod 來跑一個 Task 其實太過消耗資源, 而可以使用另一個[範例](https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/) 這個範例是用一個 Pod 執行多個任務可以降低建立多個 Pod 所花費的時間

注意的是 這邊 job 裡設定的 completions 設定代表所要完成的項目數量

也就是如果 completions 少於 Queue 內的項目代表會有項目沒被執行到

而 completions 多餘 Queue 內的項目則即使執行完所有 QUEUE 的項目, Job 仍會繼續建立 Pod 來等待QUEUE 新項目的出現

