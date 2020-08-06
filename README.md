This blog will show you how to ingest streaming(unbounded) data into SAP HANA in a fault-tolerant way with SAP Data Intelligence. We will start from a simple solution and gradually improve it until done. 

### 1. A simple data ingestion pipeline
The below figure shows a simple example pipeline provided by SAP Data Intelligence. The data generator produces sensor data as a CSV string. The data is loaded into SAP HANA and in parallel sent to a terminal, which you can use to see the generated data.

![](images/simpleIngestion.png)

Everything looks good. But what if the graph failed for some unexpected reason? We can not just blindly take the assumption that everythings works fine in a distributed environment. For example, what if there is a temporal network partition which cause the connection to HANA database timeout? The pipeline obviously will be dead. In real world, the sensor and the HANA database usually are separate compoents not coupled together like this example which means the sensor still emitting the data regardless of the failed pipeline. Are all these newly emitted data get lost? There are a lot of questions come to mind. We will look how to improve the pipeline to address those issues.

### 2. Durable messages
The first step is to decouple the message producer from the message consumer. The failed consumer should not effect the producer. The producer will continually emitting data even the consumer failed for whatever reason. Thus, the data should be persisted somewhere for consumers to ingest when they get recoverd some time later.

A good choice is to use a Log-based message broker which retains messages on disk. Apache Kafka is a message broker which persists the messages it receives. It provides a total ordering of the messages inside each partition. It also maintaits the last committed message offset.

Now we modfiy the graph to add Kafka into the pipline. This time we want to separate the message producer and the message consumer into two different graphs as we do not want to stop the producer when the consumer graph failed.

![](images/producer.png)

![](images/consumer.png)
