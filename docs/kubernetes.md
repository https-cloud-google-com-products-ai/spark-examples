Get Started with XGBoost4J-Spark on Kubernetes
==============================================
This is a getting started guide to deploy XGBoost4J-Spark package on a Kubernetes cluster. At the end of this guide, the reader will be able to run a sample Apache Spark  XGBoost application on NVIDIA GPU Kubernetes cluster.

Prerequisites
-------------
* Apache Spark 2.3+
    * Apache Spark 2.4+ if running spark-submit in [client mode](https://spark.apache.org/docs/latest/running-on-kubernetes.html#client-mode) or utilizing [Kubernetes volumes](https://spark.apache.org/docs/latest/running-on-kubernetes.html#using-kubernetes-volumes)
* Hardware Requirements
    * NVIDIA Pascal™ GPU architecture or better
    * Multi-node clusters with homogenous GPU configuration
* Software Requirements
    * Ubuntu 16.04/CentOS
    * NVIDIA driver 410.48+
    * CUDA V10.0/9.2
    * NCCL 2.4.7
* [Kubernetes 1.6+ cluster with NVIDIA GPUs](https://docs.nvidia.com/datacenter/kubernetes/index.html)
    * See official [Spark on Kubernetes](https://spark.apache.org/docs/latest/running-on-kubernetes.html#prerequisites) instructions for detailed spark-specific cluster requirements
* kubectl installed and configured in the job submission environment
    * Required for managing jobs and retrieving logs


Build a GPU Spark Docker Image
------------------------------
Build a GPU Docker image with Spark resources in it, this Docker image must be accessible by each node in the Kubernetes cluster.

1. Locate your Spark installations. If you don't have one, you can [download](https://spark.apache.org/downloads.html) from Apache and unzip it.
2. `export SPARK_HOME=<path to spark>`
3. [Download the Dockerfile](https://github.com/rapidsai/spark-examples/Dockerfile) into `${SPARK_HOME}`
4. __(OPTIONAL)__ install any additional library jars into the `${SPARK_HOME}/jars` directory
    * Most public cloud file systems are not natively supported -- pulling data and jar files from S3, GCS, etc. require installing additional libraries
5. Build and push the docker image


```
export SPARK_HOME=<path to spark>
export SPARK_DOCKER_IMAGE=<gpu spark docker image repo and name>
export SPARK_DOCKER_TAG=<spark docker image tag>

pushd ${SPARK_HOME}
wget https://github.com/rapidsai/spark-examples/raw/master/Dockerfile

# Optionally install additional jars into ${SPARK_HOME}/jars/

docker build . -t ${SPARK_DOCKER_IMAGE}:${SPARK_DOCKER_TAG}
docker push ${SPARK_DOCKER_IMAGE}:${SPARK_DOCKER_TAG}
popd
```


Get Application Jar and Dataset
-------------------------------
1. Jar: Please build the sample_xgboost_apps jar with dependencies as specified in the [README](https://github.com/rapidsai/spark-examples#example-app-jars)
2. Dataset: https://rapidsai-data.s3.us-east-2.amazonaws.com/spark/mortgage.zip

Place the required jar and dataset in a local directory. In this example the jar is in the `xgboost4j_spark/jars` directory, and the `mortgage.zip` dataset was unzipped in the `xgboost4j_spark/data` directory. 

```
[xgboost4j_spark]$ find . -type f -print|sort
./data/mortgage/csv/test/mortgage_eval_merged.csv
./data/mortgage/csv/train/mortgage_train_merged.csv
./jars/sample_xgboost_apps-0.1.4-jar-with-dependencies.jar
```

Make sure that data and jars are accessible by each node of the Kubernetes cluster via [Kubernetes volumes](https://spark.apache.org/docs/latest/running-on-kubernetes.html#using-kubernetes-volumes), on cluster filesystems like HDFS, or in [object stores like S3 and GCS](https://spark.apache.org/docs/2.3.0/cloud-integration.html). Note that using [application dependencies](https://spark.apache.org/docs/latest/running-on-kubernetes.html#dependency-management) from the submission client’s local file system is currently not yet supported.


Save Kubernetes Template Resources
----------------------------------
When using Spark on Kubernetes the driver and executor pods can be launched with pod templates. In the XGBoost4J-Spark use case, these template yaml files are used to allocate and isolate specific GPUs to each pod. The following is a barebones template file to allocate 1 GPU per pod.

```
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: gpu-example
      resources:
        limits:
          nvidia.com/gpu: 1
```

This 1 GPU template file should be sufficient for all XGBoost jobs because each executor should only run 1 task on a single GPU. Save this yaml file to the local environment of the machine you are submitting jobs from, you will need to provide a path to it as an argument in your spark-submit command. Without the template file a pod will see every GPU on the cluster node it is allocated on and can attempt to execute using a GPU that is already in use -- causing undefined behavior and errors.


Launch GPU Mortgage Example
---------------------------
Variables required to run spark-submit command:

```
# Variables dependent on how data was made accessible to each node
# Make sure to include relevant spark-submit configuration arguments
# location where data was saved
export DATA_PATH=<path to data directory> 

# location where the required jar was saved
export JARS_PATH=<path to jars directory>

# Variables independent of how data was made accessible to each node
# kubernetes master URL, used as the spark master for job submission
export SPARK_MASTER=<k8s://ip:port or k8s://URL>

# local path to the template file saved in the previous step
export TEMPLATE_PATH=${HOME}/gpu_executor_template.yaml

# spark docker image location
export SPARK_DOCKER_IMAGE=<spark docker image repo and name>
export SPARK_DOCKER_TAG=<spark docker image tag>

# kubernetes service account to launch the job with
export K8S_ACCOUNT=<kubernetes service account name>

# spark deploy mode, cluster mode recommended for spark on kubernetes
export SPARK_DEPLOY_MODE=cluster

# run a single executor for this example to limit the number of spark tasks and
# partitions to 1 as currently this number must match the number of input files
export SPARK_NUM_EXECUTORS=1

# spark driver memory
export SPARK_DRIVER_MEMORY=4g

# spark executor memory
export SPARK_EXECUTOR_MEMORY=8g

# example class to use
export EXAMPLE_CLASS=ai.rapids.spark.examples.mortgage.GPUMain

# XGBoost4J example jar
export JAR_EXAMPLE=${JARS_PATH}/sample_xgboost_apps-0.1.4-jar-with-dependencies.jar

# tree construction algorithm
export TREE_METHOD=gpu_hist
```


Run spark-submit:

```
${SPARK_HOME}/bin/spark-submit                                                          \
  --master ${SPARK_MASTER}                                                              \
  --deploy-mode ${SPARK_DEPLOY_MODE}                                                    \
  --class ${EXAMPLE_CLASS}                                                              \
  --conf spark.executor.instances=${SPARK_NUM_EXECUTORS}                                \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=${K8S_ACCOUNT}         \
  --conf spark.kubernetes.container.image=${SPARK_DOCKER_IMAGE}:${SPARK_DOCKER_TAG}     \
  --conf spark.kubernetes.driver.podTemplateFile=${TEMPLATE_PATH}                       \
  --conf spark.kubernetes.executor.podTemplateFile=${TEMPLATE_PATH}                     \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark                  \
  ${JAR_EXAMPLE}                                                                        \
  -trainDataPath=${DATA_PATH}/mortgage/csv/train/mortgage_train_merged.csv              \
  -evalDataPath=${DATA_PATH}/mortgage/csv/test/mortgage_eval_merged.csv                 \
  -format=csv                                                                           \
  -numWorkers=${SPARK_NUM_EXECUTORS}                                                    \
  -treeMethod=${TREE_METHOD}                                                            \
  -numRound=100                                                                         \
  -maxDepth=8                                                                   

```

Retrieve the logs using the driver's pod name that is printed to `stdout` by spark-submit 
```
export POD_NAME=<kubernetes pod name>
kubectl logs -f ${POD_NAME}
```

In the driver log, you should see timings* (in seconds), and the RMSE accuracy metric:
```
--------------
==> Benchmark: Elapsed time for [train]: 29.642s
--------------

--------------
==> Benchmark: Elapsed time for [transform]: 21.272s
--------------

------Accuracy of Evaluation------
0.9874184013493451
```

\* Kubernetes logs may not be nicely formatted since `stdout` and `stderr` are not kept separately

\* The timings in this Getting Started guide are only illustrative. Please see our [release announcement](https://medium.com/rapids-ai/nvidia-gpus-and-apache-spark-one-step-closer-2d99e37ac8fd) for official benchmarks.

