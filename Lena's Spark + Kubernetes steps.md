## Instructions:

There are many ways to run this job. These instructions are based on what I've done to run it using Spark's option for Kubernetes sc. There are more options!

### Packaged current Scala project in a `jar` file

```
sbt assembly
```

### Created a Kubernetes cluster and connected to it

```
az group create --name mySparkCluster --location eastus

az aks create --resource-group mySparkCluster --name mySparkCluster --node-vm-size Standard_D3_v2

az aks get-credentials --resource-group mySparkCluster --name mySparkCluster
```

### Cloned and built Spark source

```
git clone -b branch-2.3 https://github.com/apache/spark

cd spark

./build/mvn -Pkubernetes -DskipTests clean package
```

## Copied Movie Scripts under $SPARK_HOME/data directory

```
cp /path/to/got-spark-recommender/moviescript $SPARK_HOME/data/moviescript
```

## Copied a `jar` under $SPARK_HOME directory

```
cp /path/to/got-spark-recommender/target/scala-2.11/GoTEpisodeRecommender-assembly-0.1.0.jar... $SPARK_HOME/...
```

## Added a line to Dockerfile

Open `Dockerfile` under `$SPARK_HOME/resource-managers/kubernetes/docker/src/main/dockerfiles/spark/Dockerfile`

Add a line to add a `jar` file:
```
...
WORKDIR /opt/spark/work-dir

ADD GoTEpisodeRecommender-assembly-0.1.0.jar /opt/spark/jars/GoTEpisodeRecommender-assembly-0.1.0.jar

ENTRYPOINT [ "/opt/entrypoint.sh" ]
```

Note: dependencies can be either pre-mounted into the Spark container image itself, or publicly accessible (not necessarily using Azure storage for it).

### Built and pushed Spark docker image to my Dockerhub repo

```
./bin/docker-image-tool.sh -r lenadroid -t recommendations-first-season build

./bin/docker-image-tool.sh -r lenadroid -t recommendations-first-season push
```

# Submission of a Spark job

```
kubectl proxy
```

### Let's run a job!

```
./bin/spark-submit  \
--master k8s://http://127.0.0.1:8001  \
--name got-recommender  \
--deploy-mode cluster  \
--class org.apache.spark.examples.GoTEpisodeRecommender  \
--conf spark.executor.instances=2  \
--conf spark.kubernetes.container.image=lenadroid/spark:recommendations-first-season \
local:///opt/spark/jars/GoTEpisodeRecommender-assembly-0.1.0.jar stark
```