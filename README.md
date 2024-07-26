Hail on Kubernetes
====
This repository contains code do build docker images that allows to scale Hail applications using Kubernetes. 
It also provides some tools and examples.

# Goal
- Running hail in a k8s cluster.
- Use an object store S3 compatible (Minio, CephFS) with hail program

# Problem 
Hail requires :
- Java JDK 11
- Python 3.9+

Spark natives image are built upon eclipse-temurin:11.0.22_7-jdk-focal image, which include :
- Python 3.8
- A JRE instead of a JDK

# Solution 
Build a new image from  eclipse-temurin:11.0.22_7-jdk-jammy, install spark (by copying Dockerfile from spark repo) and 
install hail and required dependencies.

# How to build 
```
docker build --platform linux/amd64 -t ferlabcrsj/hail docker/hail/
docker build --platform linux/amd64 -t ferlabcrsj/hail-jupyter docker/hail-jupyter/
```

# How to test :

## With docker run :
This is a simple test to verify our image.  
```
docker run -ti --platform linux/amd64 ferlabcrsj/hail python3 -c 'import hail as hl; hl.balding_nichols_model(3, 1000, 1000).show()'
```

## Within a Kubernetes cluster :

Start and load our built images :
```
minikube start
minikube image load ferlabcrsj/hail
minikube image load ferlabcrsj/hail-jupyter
```

Create role and service account for spark
```
kubectl apply -f yaml/access.yaml
```

Create pods and services for Hail + Jupyter + Spark master :
```
kubectl apply -f yaml/master.yaml
```

To get spark-master pod name :
```
kubectl get po -o name --no-headers=true  | grep spark-master
```

### Client mode
This mode start multiple pod for the executors, but driver is embedded in master node. It requires to configure an headless service for master node.
This solution is used for notebooks (Jupyter, Zeppelin).
```
spark_master_pod=$( k get po -o name --no-headers=true  | grep spark-master)
kubectl exec -ti $spark_master_pod  -- /opt/spark/bin/spark-submit \
    --master k8s://https://kubernetes.default.svc \
    --conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
    --conf spark.kryo.registrator=is.hail.kryo.HailKryoRegistrator \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
    --conf spark.kubernetes.container.image=ferlabcrsj/hail \
    --conf spark.kubernetes.container.image.pullPolicy=Never \
    --conf spark.driver.host=spark-master \
    --conf spark.driver.port=8002 \
    --conf spark.blockManager.port=8001 \
    local:///opt/spark/work-dir/test.py
```

Expected Output :
```
24/07/26 13:52:46 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Initializing Hail with default parameters...
Running on Apache Spark version 3.5.1
SparkUI available at http://spark-master:4041
Welcome to
     __  __     <>__
    / /_/ /__  __/ /
   / __  / _ `/ / /
  /_/ /_/\_,_/_/_/   version 0.2.132-678e1f52b999
LOGGING: writing to /home/spark/hail-20240726-1352-0.2.132-678e1f52b999.log
2024-07-26 13:53:07.687 Hail: INFO: balding_nichols_model: generating genotypes for 3 populations, 1000 samples, and 1000 variants...
+---------------+------------+------+------+------+------+          (3 + 1) / 4]
| locus         | alleles    | 0.GT | 1.GT | 2.GT | 3.GT |
+---------------+------------+------+------+------+------+
| locus<GRCh37> | array<str> | call | call | call | call |
+---------------+------------+------+------+------+------+
| 1:1           | ["A","C"]  | 0/1  | 0/0  | 0/1  | 0/0  |
| 1:2           | ["A","C"]  | 1/1  | 1/1  | 1/1  | 1/1  |
| 1:3           | ["A","C"]  | 0/1  | 0/1  | 1/1  | 0/1  |
| 1:4           | ["A","C"]  | 0/1  | 0/0  | 0/1  | 0/0  |
| 1:5           | ["A","C"]  | 0/1  | 0/1  | 0/1  | 0/0  |
| 1:6           | ["A","C"]  | 1/1  | 1/1  | 1/1  | 1/1  |
| 1:7           | ["A","C"]  | 0/1  | 1/1  | 1/1  | 0/1  |
| 1:8           | ["A","C"]  | 0/0  | 0/1  | 0/1  | 0/0  |
| 1:9           | ["A","C"]  | 0/0  | 0/0  | 0/0  | 0/0  |
| 1:10          | ["A","C"]  | 0/0  | 0/0  | 0/1  | 0/1  |
+---------------+------------+------+------+------+------+
showing top 10 rows
showing the first 4 of 1000 columns    
```

You can see that executors are created :
```
kubectl get po
...
NAME                              READY   STATUS      RESTARTS   AGE
hail-1919b290ef89c68d-exec-1      1/1     Running     0          4s
hail-1919b290ef89c68d-exec-2      1/1     Running     0          2s
spark-master-87999d7db-zg72s      1/1     Running   0          7m
```
Executors will be automatically removed at the end of the process.

### Cluster mode
This mode start one pod for the driver and multiple pod for the executors. 
This is preferred solution for dynamic process (like ETL) because no additional service configuration is required.  

```
spark_master_pod=$( k get po -o name --no-headers=true  | grep spark-master)
kubectl exec -ti $spark_master_pod  -- /opt/spark/bin/spark-submit \
    --master k8s://https://kubernetes.default.svc \
    --deploy-mode cluster \
    --conf spark.serializer=org.apache.spark.serializer.KryoSerializer \
    --conf spark.kryo.registrator=is.hail.kryo.HailKryoRegistrator \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
    --conf spark.kubernetes.container.image=ferlabcrsj/hail \
    --conf spark.kubernetes.container.image.pullPolicy=Never \
    local:///opt/spark/work-dir/test.py
```

You can see that driver and executors are created :
```
kubectl get po
...
NAME                              READY   STATUS    RESTARTS   AGE
hail-c1cab290ef8b4e05-exec-1      1/1     Running   0          2s
hail-c1cab290ef8b4e05-exec-2      1/1     Running   0          1s
minio                             1/1     Running   0          2m38s
spark-master-87999d7db-zg72s      1/1     Running   0          7m
test-py-84c92790ef8b233a-driver   1/1     Running   0          13s
```
Executors will be automatically removed at the end of the process. But the driver will be kept until manually deleted.
You can then check driver logs :

```
kubectl logs test-py-84c92790ef8b233a-driver
```
Expected Output :
```
24/07/26 14:56:38 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Initializing Hail with default parameters...
Running on Apache Spark version 3.5.1
SparkUI available at http://test-py-84c92790ef8b233a-driver-svc.default.svc:4040
Welcome to
     __  __     <>__
    / /_/ /__  __/ /
   / __  / _ `/ / /
  /_/ /_/\_,_/_/_/   version 0.2.132-678e1f52b999
LOGGING: writing to /opt/spark/work-dir/hail-20240726-1456-0.2.132-678e1f52b999.log
2024-07-26 14:56:57.175 Hail: INFO: balding_nichols_model: generating genotypes for 3 populations, 1000 samples, and 1000 variants...
+---------------+------------+------+------+------+------+
| locus         | alleles    | 0.GT | 1.GT | 2.GT | 3.GT |
+---------------+------------+------+------+------+------+
| locus<GRCh37> | array<str> | call | call | call | call |
+---------------+------------+------+------+------+------+
| 1:1           | ["A","C"]  | 0/1  | 0/0  | 0/1  | 0/0  |
| 1:2           | ["A","C"]  | 1/1  | 1/1  | 1/1  | 1/1  |
| 1:3           | ["A","C"]  | 0/1  | 0/1  | 1/1  | 0/1  |
| 1:4           | ["A","C"]  | 0/1  | 0/0  | 0/1  | 0/0  |
| 1:5           | ["A","C"]  | 0/1  | 0/1  | 0/1  | 0/0  |
| 1:6           | ["A","C"]  | 1/1  | 1/1  | 1/1  | 1/1  |
| 1:7           | ["A","C"]  | 0/1  | 1/1  | 1/1  | 0/1  |
| 1:8           | ["A","C"]  | 0/0  | 0/1  | 0/1  | 0/0  |
| 1:9           | ["A","C"]  | 0/0  | 0/0  | 0/0  | 0/0  |
| 1:10          | ["A","C"]  | 0/0  | 0/0  | 0/1  | 0/1  |
+---------------+------------+------+------+------+------+
showing top 10 rows
showing the first 4 of 1000 columns
```

# Minio

To install minio in k8s cluster :
```
kubectl apply -f yaml/minio-dev.yaml
```

Then to be able to access minio 
```
kubectl port-forward minio 9090:9090 9000:9000
```

Then you can configure minio client (or any s3 compatible client) :
```mc alias set minikube http://127.0.0.1:9000 minioadmin minioadmin```

Create a hail bucket :
```
mc mb minikube/hail
```

To copy files into minio, you can use `mc copy`. For instance:
```
mc cp ~/Desktop/gvcf/* minikube/hail/gvcf/
```

# Jupyter 
To connect to jupyter
```
spark_master_pod=$( k get po -o name --no-headers=true  | grep spark-master)
kubectl port-forward $spark_master_pod 8888:8888 4040:4040
```

Then check logs :
```
spark_master_pod=$( k get po -o name --no-headers=true  | grep spark-master)
kubectl logs $spark_master_pod
....
[I 2024-07-25 17:37:42.003 ServerApp] Jupyter Server 2.14.2 is running at:
[I 2024-07-25 17:37:42.003 ServerApp] http://localhost:8888/lab?token=xxx
[I 2024-07-25 17:37:42.003 ServerApp]     http://127.0.0.1:8888/lab?token=xxx
...
```
Then copy/paste the link displayed in the logs into your browser.

You can then import notebook in `notebook/` directory to play with hail and minio on k8s.
