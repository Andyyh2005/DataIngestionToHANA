# DataIngestionToHANA

This blog will show you how to ingest streaming(unbounded) data into SAP HANA in a fault-tolerant way with SAP Data Intelligence. We will start from a simple solution and gradually improve it until done. 

### 1. A simple data ingestion pipeline
The below figure shows a simple example pipeline provided by SAP Data Intelligence. The data generator produces sensor data as a CSV string. The data is loaded into SAP HANA and in parallel sent to a terminal, which you can use to see the generated data.
