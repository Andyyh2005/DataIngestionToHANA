This blog will show you how to ingest stream(unbounded) data into SAP HANA in a fault-tolerant way using SAP Data Intelligence. We will start from a simple solution and improve it gradually until the processing meet the demand of our expectation. 

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

![](images/consumerAtMostOnce.png)

For debug and testing purpose, I added more operators into the consumer graph:
- Message operator labeled as "Processing Data": A stream processor simply sleep some time to mimic a time-consuming message processing and forward the receiving message to downstream operator. Its processing code is as below:
```
function sleep(millisecondsToWait) {
  var now = new Date().getTime();
  while(new Date().getTime() < now + millisecondsToWait) {}
}

function onInput(ctx,s) {
    sleep(5000);
    $.output(s);
}

$.setPortCallback("input",onInput);
```
- "Terminal" operator: we will use it to send a signal indicating an unexpected error occured to the downstream operator.
- Message Operator labeled as "Simulate Error" receieves message from HANA Client and forward it to downstream opeator. It also has a debug port used for receieving a terminal signal. After receieve the signal, the graph will be dead if it receieves a subsequent incoming message from HANA Client. Its code is as below:

```
var terminate = false;

function onInput(ctx,s) {
    if(terminate) {
        $.fail("unexpected value received");
    }
    $.output(s);
}

function onDebug(ctx, s) {
    terminate = true;
}

$.setPortCallback("input",onInput);
$.setPortCallback("debug",onDebug);
```
- Wiretap opeator simply trace the messages ingested into HANA.

Next we want to see what kind of message delivery guarantee the pipeline can provide with different configuration and settings.