
# Registration center

## Responsibility
### Service registration
### Service discovery
### Heartbeat detection 

## CP or AP model?
* Service registration: Si = F(ServiceName)
	- ServiceName is a lookup parameter
	- Si is the corresponding service' IP:Port list

### Case study 
* Setup:
	- Service A is calling Service B
	- Service A has two deployments A1 and A2
	- Service B has one deployment with IP port from IP1 -> IP10
	- A1 deployment only gets IP1 -> IP9
	- A2 deployment only gets IP2 -> IP10
* If using CP model
	- Then A1 and A2 are both not available
* If using AP model
	- Then the only impact is that the traffic distribution on IP1 -> IP10 is not even. 
* Should use AP model

## Implementation

### Comparison (In Chinese)
![Comparison](./images/serviceDiscovery_comparison.png)

### Zookeeper?
* It is becoming popular because it is the default registration center for Dubbo framework. 

#### Definition
* Apache Zookeeper is an effort to develop and maintain an open-source server which enables highly reliable distributed coordination.
* Zookeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. 
* Zookeeper is an open source implementation of Google Chubby.
* In essence, it is a file system with notification capabilities. 

#### Functionality
* Leader election: Two nodes watch the same node. 
* Configuration management
* Distributed lock
* Service registration
* Service discovery

#### Limitations
##### CP model
* Zookeeper is in essence a CP model, not an AP model. 
	* [Why you shouldn't use Zookeeper for service discovery](https://medium.com/knerd/eureka-why-you-shouldnt-use-zookeeper-for-service-discovery-4932c5c7e764)
* Example
	* Setup: 
		- There are five nodes deployed in three clusters
		- ZK1/ZK2 in Cluster 1, ZK3/ZK4 in Cluster 2, ZK5 in Cluster 3. 
		- Suddenly there is a network partition between Cluster1/2 and Cluster3. 
		- Within each cluster, there is an application cluster. Application service A is calling service B. 
	* If using CP model
		- ZK5 is not readable or writable. All clients connected to Cluster3 will fail. 
		- Then Service B cannot be scaled up. 
		- Then Service A cannot be rebooted.
	* Should use AP model

##### Scalability
* All writes will come to master node, then it will be replicated to the majority of slave nodes synchronously (two phase commit is used for replication). 
	- Possible solution: Split Zookeeper cluster according to the business functionalities. Different business units will not talk to each other. 
	- Limitation: Due to the reorg and evolution, different business units will need to talk to each other. 

##### Storage
* Since Zookeeper is based on ZAB protocol. ZAB will maintain a commit log on each node which records every write request. On a regular basis the in-memory records will be dumped to disk. This means that the entire history of write change will be recorded. 
* From the perspective of registration center, it does not need to access history of node changes
	- It only needs to access information such as epoch number, weight of different nodes

##### Service health check
* Zookeeper is using Keep-Alive heart-beat message functionality to detect the liveness of the node. 
	- When working thread within business logic unit dies, the Zookeeper will not be informed. 
* Ideal solution:
	- The business unit should send a health info to an Zookeeper healthcheck API. 

##### Nonfault tolerant native client
* Native Zookeeper client does not have resiliency built-in. It won't fall back to client snapshot/cache when Zookeeper registration center is down. 

#### Applicable scenarios
* It is the king of coordination for big data. These scenarios don't require high concurrency support. 
	- e.g. kafka will only use Zookeeper during leader election scenarios. 
	- e.g. Hadoop will need Zookeeper for map-reduce coordination scenarios.

### DIY
#### Requirements
* Cross language support
* Use instructions to control the cluster
	- e.g. Proactively ask the cluster to downgrade a specific service with an instruction
	- e.g. Rate limiter
	- e.g. Circuit breaker
* Store information of registration nodes 
	- e.g. Node type: virtual machine, docker node, physical node
	- e.g. Node weight
	- e.g. Grouping

#### Standalone design
##### Service management platform
* Responsible for 
	- Initiating all commands
	- Service discovery
* Has its own Redis/MySQL center
	- So that not each time a service discovery is needed, the service management platform needs to talk to registration center.

##### Business logic unit
* Business logic unit maintains a TCP long-lived connection to registration center
	- Business logic unit talks to registration center: Service discovery
	- Registration center talks to business logic unit: Send command ???

##### Gateway
* Gateway maintains a TCP long-lived connection to registration center
	- Gateway talks to registration center: Service discovery
	- Registration center talks to gateway: Send command
* Gateway maintains a HTTP connection to service management platform
	1. Gateway asks IP addresses of a business logic unit name from service management platform
	2. Service management platform will talk to registration center for the IP address 

##### Registration center
* Responsible for service registration, service discovery, instruction execution

###### Registration center client
* Service management platform uses registration center client to talks to registration center. 

###### Registration center plugin
* Def: A SDK/Jar 
* Exist within business logic unit / Gateway: 
	- Business logic unit uses this to register service, discover service and receive command from registration center

##### Flowchart

###### Instruction pushdown
* Instructions
	- Disable certain business logic unit
	- Downgrade, circuit breaker, rate limit
	- Configuration 
* A downgrade instruction to downgrade business logic unit
	1. Registration center receives the downgrade command from service management platform via registration center client
	2. Registration center sends the downgrade command to gateway via registration center plugin
	3. Gateway terminates some connection 

###### Service stop
* There are two business logic units A and B. A dies
	1. Registration center knows the event from TCP long lived connection
	2. Registration center sends the event to service management center

###### Service start
* A new service instance starts up
	1. Business logic sends the info to service registration center via long lived connection.
	2. Registration center sends the info to gateway via long lived connection

###### Service lookup
* Gateway sends the info to service management platform via HTTP connection
* Service management platform talks to service registration center. Passing in the business logic name, wanting the business logic IP address. 

##### Comparison (In Chinese)
* ![Comparison](./images/serviceRegistration_comparison.png)

#### Scalable design
##### Registration center 
* Multiple registration center deployed
	- Different gateways/business logic units are connected to different registration center
* What if a pushdown instruction arrives at the wrong registration center
	- [Gossip protocol](./gossipProtocol.md) to discover between different registration centers
		+ Cons: Long message delay / Forward redundancy
		+ Pros: Suitable for large number of clusters. Used in P2P, Redis cluster, Consul
	- An open source implementation for Gossip https://github.com/scalecube/scalecube-cluster

##### Registration center plugin
* Registration center plugin will get a list of IP addresses of registration center. 
	- If all connections fail, then alert
* DNS domain 
	- ccsip.nx.com


#### Question
* Why so many different types of connection
	- Plugin vs Client vs HTTP vs TCP Long-lived connection
