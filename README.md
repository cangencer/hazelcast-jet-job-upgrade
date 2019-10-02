# Jet Job Upgrade Demo

This is a demonstration of operational features of Hazelcast Jet:

- Job Upgrades 
- Cluster Elasticity
- Lossless Cluster Restart
- Management tools: Management Center and `jet.sh`


## Data Pipeline

Jet job analyses trade events to computes moving average price and total volume of trades over various time windows.

Trades are randomly generated and stored to a Kafka topic (10k trades/sec). Jet job draws the trades from Kafka and 
processes it. Results are streamed to InfluxDB and plotted as line charts in a Grafana dashboard. 

## Job Upgrades

Jet allows you to upgrade a running job while preserving its state. This is useful for bug fixing or 
to deploy new version of the application.

We'll add new calculations to the running job. First version of the job produces just average price over one second 
window. Upgraded version them adds one minute average and one second volume of trades.

The one second average calculation keeps running after the upgrade without data loss or data corruption. 

To learn more please see the [reference manual](https://docs.hazelcast.org/docs/jet/latest/manual/#job-upgrade).

## Cluster Elasticity

Jet cluster is elastic. It keeps processing data without loss even if a node fails, and you can add more nodes that 
immediately start sharing the computation load.  The streaming jobs tend to be long-running tasks. The elasticity of Jet 
cluster allows scaling up and down with the load to cover load spikes and to prevent overprovisioning.

The demo shows how to add or remove the node from the running cluster. 

## Lossless Cluster Restart

The elasticty feature described in the previous section is entirely RAM-based, the data is replicated in cluster memory.
Given the redundancy present in the cluster, this is sufficient to maintain a running cluster across single-node failures 
(or multiple-node, depending on the backup count), but it doesn’t cover the case when the entire cluster must shut down.

[Lossless Restart](https://docs.hazelcast.org/docs/jet/latest/manual/#configure-lossless-cluster-restart-enterprise-only) 
feature allow you to gracefully shut down the cluster at any time and have all the jobs preserved 
consistently. After you restart the cluster, Jet automatically restores the data and resumes the jobs.

## Management Center and jet.sh

[Management Center](https://docs.hazelcast.org/docs/jet/latest/manual/#management-center) is an UI to monitor and manage
 Jet. `jet.sh` is a [command-line tool](https://docs.hazelcast.org/docs/jet/latest/manual/#command-line) to deploy and 
 manage jobs. 
 
We'll use both to deploy and upgrade new jobs and control the cluster. 


# Code Structure

The `TradeAnalyser` class contains the analytical job. It uses `JetBootstrap` to 
[submit the job](https://docs.hazelcast.org/docs/jet/latest/manual/#jet-submit) to the cluster from the command line. 

The `TradeProducer` class generates Trades and sends them to Kafka.

# Prerequisites

## Hazelcast Jet Enterprise

Download [Hazelcast Jet 3.x](https://hazelcast.com/download/) and unzip it.

Insert the enterprise license key to `${JET_HOME}/config/hazelcast.xml`. 
You can get a trial license key from [Hazelcast](https://hazelcast.com/download/).

Enable Lossless Restart in `${JET_HOME}/config/hazelcast-jet.xml` by setting ```<lossless-restart-enabled>true</lossless-restart-enabled>```.

Start Jet cluster member:

```bash
${JET_HOME}/bin/jet-start.sh 
```

## Management Center

Management Center is distributed with Hazelcast Jet Enteprise. One has to configure the enterprise licesnse to 
use Management Center for clusters larger than one node.

Insert the license key to `${JET_HOME}/hazelcast-jet-management-center/application.properties`. Use the same license key 
as for Hazelcast Jet.  

Start Management Center for Jet:

```bash
${JET_HOME}/hazelcast-jet-management-center/jet-management-center.sh
```

Navigate your web browser to the Management Center (<http://localhost:8081/> by default) to see the running Jet cluster:

![Management Center](/img/management-center.png)  

## Kafka

Install [Apache Kafka](https://kafka.apache.org/) 1.x or 2.x.

Create the `trades` topic in Kafka:

```bash
kafka-topics --create --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1  --topic trades
```

Start the Kafka server.

## InfluxDB

Install [InfluxDB](https://www.influxdata.com/products/influxdb-overview/) 1.x.

Start InfluxDB.

Create the `trades` database using the `influx` tool:

```bash
influx
CREATE DATABASE trades
```

## Grafana

Install [Grafana](https://grafana.com/) 6.x

Navigate your web browser to the Grafana UI and create an InfluxDB data source named InfluxDB, with default URL and 
default password (root:root).

Create the dashboard by importing `grafana/trade-analysis-dashboard.json`

You should see a screen similar to this:

![Grafana dashboard](/img/grafana-default-screen.png)


# Building the Application

To build and package the application, run:

```bash
mvn clean package
```

# Running the Application



## Start producing Trades

This shortcut starts the TradeProducer

```bash
mvn exec:java
```

It keeps generating trades at the rate of 10k trades/sec.

## Start the Job

Deploy TradeAnalyser Jet job:

```bash
${JET_HOME}/bin/jet.sh submit -n TradeAnalyserVersion1 target/trade-analyser-3.0-SNAPSHOT.jar
```

Job starts running as soon as it's deployed.

The Grafana dashboard now plots the one second moving averages for 3 data lines (3 trade symbols):

![First Job version](/img/job-version-1.png)

## Upgrade the Job

To demonstrate job upgrade feature, change the Trade Analyser code by enabling two more computations. Uncomment 
following lines in the `src/main/java/TradeAnalyser.java`:

```java
//        StreamStage<Point> avg1m = grouped
//                .window(WindowDefinition.sliding(60_000, 1_000))
//                .aggregate(AggregateOperations.averagingLong(Trade::getPrice))
//                .setName("avg-1m")
//                .map(res -> mapToPoint("avg_1m", res))
//                .setName("map-to-point");
//
//        StreamStage<Point> vol1s = source
//                .window(WindowDefinition.tumbling(1_000))
//                .aggregate(AggregateOperations.summingLong(Trade::getQuantity))
//                .map(res -> mapToPoint("vol_1s", res))
//                .setName("vol-1s")
//                .setName("map-to-point");
```

Also, the sink definition must be changed in the code to sink results of added computations to InfluxDB. 
Disable original sink and enable `drainTo` that includes the new computations: 

```java
//        avg1s.drainTo(influxDbSink);
p.drainTo(influxDbSink, avg1m, avg1s, vol1s);
```

Build the changed TradeAnalyser:

```bash
mvn clean package
```

Upgrade the running Jet job with the new verison:
```bash
${JET_HOME}/bin/jet.sh save-snapshot --cancel TradeAnalyserVersion1 SavePoint1
${JET_HOME}/bin/jet.sh submit -n TradeAnalyserVersion2 -s SavePoint1 target/trade-analyser-3.0-SNAPSHOT.jar

``` 

You'll see new data points in the chart as soon as the new job version is deployed.

![First Job version](/img/job-version-2.png) 

You may also inspect the execution graph of the upgraded job verison in the Management Center. 

## Lossless Restart

Shutdown the cluster gracefully. Gracefull shutdown waits for cluster state to be saved to the disk. 

```bash
${JET_HOME}/bin/cluster.sh -o shutdown -g jet - P jet-pass
```

You can inspect the Grafana dashboard - no new data is comming.

Start the cluster node again. The job is restarted without a data loss. The computation is restored where it left off.
Jet quickly catches up with the new trades that were added to Kafka while Jet was off.

```bash
${JET_HOME}/bin/jet-start.sh 
```

## Elastic scaling

Add a member to the running cluster:

```bash
${JET_HOME}/bin/jet-start.sh 
```

The member joins the cluster and the computation is rescaled automatically to make use of the added resources.
Check the cluster state from the command line or in the Management Center:

```bash
${JET_HOME}/bin/jet.sh cluster
```

You should see someting like:
```bash
State: ACTIVE
Version: 3.0
Size: 2

ADDRESS                  UUID               
[localhost]:5701         d7cf92d9-85b6-486d-9f48-7ac70f2147b3
[localhost]:5702         3e0842e4-0210-42c7-954f-8076c482c898
```

Now you can remove a cluster member by simply killing one of the Jet cluster processes. The fault-tolerance mechanism
kicks in and the computation continues without data loss.
