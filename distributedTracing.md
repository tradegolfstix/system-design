
# Distributed tracing
## Use case
* When bottleneck for application performance and Core service dies and result in tracing
* How to resolve:
	- Solve it by business side
	- Distributed tracing system

## History
* 2014 Google Dapper. "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"
* Alibaba EagleEye
* DaZhongDianPing CAT 
* Jingdong Hydra
* Twitter Zipkin
* Apache SkyWalking (APM - Application Performance Management)
* Pinpoint (APM)

## Requirements
* Not much change in business logic required, reduce load on business developers
* Could define the sampling rate and on-off switch
* Latency not too low

## Components
![Distributed tracing](./images/distributedTracing_OverallFlow.png)

### Generate logs
* Only needs to import jar pakcagge
* Or when starting the application, add additional parameter
	- javaagent: Which starts application in Java agent mode. It adds a pre-event interceptor and an after-event interceptor. Then performance metrics could be collected by agent. 
	- Introduced within JDK 1.5 (Byte enhancement tool)
* Send the log every 5 seconds

![Distributed tracing](./images/distributedTracing_javaAgent.png)

#### Unique Id
* TraceID
	- Generate using UUID
* SequenceID
	- Gateway calls business logic unit 1 first, then business logic unit 2. SequenceId is used to number this. Differentiate the request on the breadth level
	- Increment from 1 
* DepthID
	- Differentiate the request on the depth level
	- The number of dots in an ID

![Distributed tracing](./images/distributedTracing_IdDesign.png)

### Offline analysis
* Based on Hadoop

### Real-time analysis
* Spark/Flume performs real-time analysis for QPS, average response time
* Result is being piped into Redis

### Full data 
* Elasticsearch to search

## Pinpoint
### Supported intercepting components
![Distributed tracing](./images/distributedTracing_SupportedInterceptor.png)

### How to generate Spanner ID
* Not incremental, a random number between min(long) and max(long)

![Distributed tracing](./images/distributedTracing_SpannerID.png)
