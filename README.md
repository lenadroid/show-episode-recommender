# show-episode-recommender
A Spark job that outputs the ranked list of episode to watch based on one preference "keyword".

## What this Job is supposed to do

This Spark job is a prototype of a trivial recommendation system for TV-show episodes to watch based on your preference.

### Input requirements

* A directory with text files containing dialogie scripts for each episode of the show. Current scripts are from the 1st season of "The Game of Thrones". It is absolutely okay to replace existing files in the `moviescript` directory with dialogue scripts from the TV-show of your choice, or even a series of book scripts.
* A keyword of preference. It can be a favorite show character name, or a favorite thing that gets mentioned throughout the dialogue. Foe example, `tyrion`, `arya`, `nymeria` or `cersei` would all be valid things to have as a keyword of preference.

### Expected output

When the job is completed, it should output a ranked list of episodes (or books, or etc.) corresponding to the specified keyword of preference sorted from most preferable to least preferable. For example, in case the analysis is done for `tyrion` as a keyword of preference, and the `moviescript` directory contains dialogies for episodes of the first season of Game of Thrones, the ranked list of episodes will look similar to following:

```
1. file:/opt/spark/data/moviescript/8.The Pointy End.txt
2. file:/opt/spark/data/moviescript/6.A Golden Crown.txt
3. file:/opt/spark/data/moviescript/1.Winter is Coming.txt
4. file:/opt/spark/data/moviescript/3.Lord Snow.txt
5. file:/opt/spark/data/moviescript/5.The Wolf and the Lion.txt
6. file:/opt/spark/data/moviescript/7.You Win or You Die.txt
```

In case the keyword of preference isn't mentioned in every episode, the ranked list wil me smaller than the original list.

## Instructions

There are many ways to run this Spark job, depending on resource scheduler and other criteria. These instructions are based on what I've done to run the job using the Spark's new option for Kubernetes scheduler (starting from 2.3 release). There are more options how to run it than described here!

### Package current Scala project in a `jar` file

Clone the project and navigate into it.

```
git clone git@github.com:lenadroid/show-episode-recommender.git
cd show-episode-recommender
export RECOMMENDER=$(pwd)
```

Make a `jar`.
```
sbt assembly
```

### Create a Kubernetes cluster and connected to it

I am using AKS (Azure Container Service), but any Kubernetes cluster should work.
```
az group create --name mySparkCluster --location eastus

az aks create --resource-group mySparkCluster --name mySparkCluster --node-vm-size Standard_D3_v2

az aks get-credentials --resource-group mySparkCluster --name mySparkCluster
```

### Clone and built Spark source

```
git clone -b branch-2.3 https://github.com/apache/spark

cd spark

export SPARK_HOME=$(pwd)

./build/mvn -Pkubernetes -DskipTests clean package
```

## Copy Movie Scripts under $SPARK_HOME/data directory

Or upload them to somewhere remote where you can read it from using `sc.wholeTextFiles("...movie scripts directory address...")`.

```
cp $RECOMMENDER/moviescript $SPARK_HOME/data/moviescript
```

## Copy a `jar` under $SPARK_HOME directory

```
cp $RECOMMENDER/target/scala-2.11/GoTEpisodeRecommender-assembly-0.1.0.jar $SPARK_HOME/GoTEpisodeRecommender-assembly-0.1.0.jar
```

## Add a line to Dockerfile

Open `Dockerfile` under `$SPARK_HOME/resource-managers/kubernetes/docker/src/main/dockerfiles/spark/Dockerfile`

Add a line to add a `jar` file:
```
...
WORKDIR /opt/spark/work-dir

ADD GoTEpisodeRecommender-assembly-0.1.0.jar /opt/spark/jars/GoTEpisodeRecommender-assembly-0.1.0.jar

ENTRYPOINT [ "/opt/entrypoint.sh" ]
```

Note: dependencies can be either pre-mounted into the Spark container image itself, or remote. More details [here](https://spark.apache.org/docs/latest/running-on-kubernetes.html).

### Build and push Spark docker image to my Dockerhub repo

```
export CONTAINER_REGISTRY="..."
./bin/docker-image-tool.sh -r $CONTAINER_REGISTRY -t recommendations-first-season build

./bin/docker-image-tool.sh -r $CONTAINER_REGISTRY -t recommendations-first-season push
```

# Submission of a Spark job

Start Kubernetes proxy.
```
kubectl proxy
```

### Let's run a job!

```
$SPARK_HOME/bin/spark-submit  \
--master k8s://http://127.0.0.1:8001  \
--name got-recommender  \
--deploy-mode cluster  \
--class org.apache.spark.examples.GoTEpisodeRecommender  \
--conf spark.executor.instances=2  \
--conf spark.kubernetes.container.image=$CONTAINER_REGISTRY/spark:recommendations-first-season \
local:///opt/spark/jars/GoTEpisodeRecommender-assembly-0.1.0.jar stark
```