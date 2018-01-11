# Monitoring Kafka with Jolokia

Todays big data analytics and streaming applications often rely on [Apacha Kafka](https://kafka.apache.org/). I was also working on a medium-sized data processing engine for a customer. In the first release we didn't had sophisticated solutions in place to manage Kafka like .... But since we were producing a production release we required a simple solution to monitor our Kafka Brokers. The most simple solution I've found was Jolokia.

---

[Jolokia](https://jolokia.org/) is a JMX-HTTP bridge giving an alternative to JSR-160 connectors. It provides simple access to JMX beans via a RESTful API. This is a good fit to Kafka, as Kafka provides multiple JMX beans to monitor the brokers. Especially if your default monitoring tools rely on HTTP endpoints to check like in our case.

The following values can be obtained form Kafka via JMS (source: [server density](https://blog.serverdensity.com/how-to-monitor-kafka/)):

|Metric|Comments|
|--- |--- |
|UnderReplicatedPartitions|kafka.server: type=ReplicaManager, name=UnderReplicatedPartitions – Number of under-replicated partitions.|
|OfflinePartitionsCount|kafka.controller: type=KafkaController, name=OfflinePartitionsCount – Number of partitions without an active leader, therefore not readable nor writeable.|
|ActiveControllerCount|kafka.controller: type=KafkaController, name=ActiveControllerCount – Number of active controller brokers.|
|MessagesInPerSec|kafka.server: type=BrokerTopicMetrics, name=MessagesInPerSec – Incoming messages per second.|
|BytesInPerSec / BytesOutPerSec|kafka.server: type=BrokerTopicMetrics, name=BytesInPerSec – kafka.server: type=BrokerTopicMetrics, name=BytesOutPerSec – Incoming/outgoing bytes per second.|
|RequestsPerSec|kafka.network: type=RequestMetrics, name=RequestsPerSec, request={Produce|FetchConsumer|FetchFollower} – Number of requests per second.|
|TotalTimeMs|kafka.network: type=RequestMetrics, name=TotalTimeMs, request={Produce|FetchConsumer|FetchFollower} – Total time it takes to process a request. You can also monitor split times for QueueTimeMs, LocalTimeMs, RemoteTimeMs and RemoteTimeMs.|
|UncleanLeaderElectionsPerSec|kafka.controller: type=ControllerStats, name=LeaderElectionRateAndTimeMs – Number of disputed leader elections rate.|
|LogFlushRateAndTimeMs|kafka.log: type=LogFlushStats, name=LogFlushRateAndTimeMs – Asynchronous disk log flush and time in ms.|
|UncleanLeaderElectionsPerSec|kafka.controller: type=ControllerStats, name=UncleanLeaderElectionsPerSec – Unclean leader election rate.|
|PartitionCount|kafka.server: type=ReplicaManager, name=PartitionCount – Number of partitions on your system.|
|ISR shrink/expansion rate|kafka.server: type=ReplicaManager, name=IsrShrinksPerSec – kafka.server: type=ReplicaManager, name=IsrExpandsPerSec – When a broker goes down, ISR will shrink for some of the partitions. When that broker is up again, ISR will be expanded once the replicas have fully caught up.|
|NetworkProcessorAvgIdlePercent|kafka.network: type=SocketServer, name=NetworkProcessorAvgIdlePercent – The average fraction of time the network processors are idle.|
|RequestHandlerAvgIdlePercent|kafka.server: type=KafkaRequestHandlerPool, name=RequestHandlerAvgIdlePercent – The average fraction of time the request handler threads are idle.|
|Heap Memory Usage|Memory allocated dynamically by the Java process, Zookeeper in this case.|


To provide these value via Jolokia, execute the following steps:

  * Make sure that you have an JDK installed on your Broker's machine - Jolokia requires the JDK's `tools.jar`
  
  * Download the Jolokia JVM agent to your Broker's machine:

    ```bash
    curl -L -O http://search.maven.org/remotecontent?filepath=org/jolokia/jolokia-jvm/1.3.7/jolokia-jvm-1.3.7-agent.jar
    ```

  * Get the PID of your Broker's process

    ```bash
    ps aux | grep ${USER} | grep "kafka.Kafka" | grep -v grep | awk '{ print $2 }'
    ```

  * Now we can attach the Jolokia agent to Kafka's Java process. When starting the monitoring process make sure that you do it with exactly the same user as the monitored process, otherweise Jolokia will not be able to attach itself to the process. In my example the user who runs the Broker is `kafka`.

    ```bash
    su kafka -c "export JAVA_HOME=/opt/jdk; /opt/jdk/bin/java -jar /opt/jolokia/jolokia-jvm-1.3.7-agent.jar start ${PID} --host <YOUR_HOSTNAME>"
    ```

    Optionally you can provide the `--host` parameter to Jolokia. This usually makes sens since the default hostname to which the agent is listening is `localhost`. If you want to access the API from another machine (e.g. your monitoring) you must provide an external reachable hostname.

  * Now you're able to read the JMX bean values:

    ```bash
    curl http://<YOUR_HOSTNAME>:8778/jolokia/read/kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
    ```

    The response will look like this:

    ```json
    {
      "request":{
        "mbean":"kafka.server:name=UnderReplicatedPartitions,type=ReplicaManager",
        "type":"read"
      },
      "value":{
        "Value":0
      },
      "timestamp":1515666690,
      "status":200
    }
    ```

  * To stop Jolokia, just execute the stop command on `$PID`:

    ```bash
    su kafka -c "export JAVA_HOME=/opt/jdk; /opt/jdk/bin/java -jar /opt/jolokia/jolokia-jvm-1.3.7-agent.jar stop ${PID}"
    ```

That's it. With these simple steps you can provide the values required for monitoring Kafka easily with Jolokia.