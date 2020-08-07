## 1. A simple data ingestion pipeline
The below figure shows a simple example pipeline provided by SAP Data Intelligence. The data generator produces sensor data as a CSV string message. The message is loaded into SAP HANA and in parallel sent to a terminal, which you can use to see the generated data.

![](images/simpleIngestion.png)

It looks good if it is only for a demonstration purpose. However, in reality we cannot take the assumption that everything works just fine in a distributed environment. The pipeline execution might be dead if something unexpected occurred. For example, what if there is a temporal network partition and the HANA Client connection timeout? We need our pipeline executed in a fault-tolerant way. 

- The data generator should decouple from its consumers. And it should be able to continue generate messages regardless if the consumers failed or not. 
- Regardless of what caused the consumer failure and how long the failure lasts, it should be able to recover by picking up the data at the place where it left during the failure period instead of processing from scratch.This will save the processing time.
- Losing data or duplicated processing data will casue the derived HANA database table go permanently out of sync with the data generator. Thus message delivery must be reliable. 

Next, we will look at how to improve the pipeline to address these issues.

## 2. Durable messages
To decouple the message producers from the message consumers, the data should be persisted somewhere.

A good choice is to use a Log-based message broker which retains messages on disk. Apache Kafka is a message broker which persists the messages it receives. It also maintains the last committed message offset.

Now we modfiy the graph to add Kafka into the pipeline. This time we want to place the message producer and the message consumer into two separate graphs as we do not want to stop the producer when the consumer graph failed.

![](images/producer.png)

![](images/consumer.png)

> I also changed the terminal operator to wiretap operator which shows more message information.

Now, by incorporating Kafka into our pipeline, we are flexbilbe enough to debug and test our pipeline.

## 3. Message delivery guarantee
We want to see the different message delivery guarantees the pipeline can provide using different configuration.

Add more operators into the pipeline like below.