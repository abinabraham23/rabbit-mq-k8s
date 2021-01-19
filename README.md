# RabbitMQ-K8s
Repository includes K8s resources and simple java program (sender-receiver) to test for RabbitMQ

## Kubernetes Objects:

1. **Namespace**
Create a dedicated namespace to keep the RabbitMQ objects separately from other k8s objects in a cluster. It also helps with better management of access & permissions on the R-MQ related objects. 
`kubectl apply -f namespace.yml`

2. **RBAC**
R-MQ’s Kubernetes peer discovery plugin relies on Kubernetes API as data source. On first boot, every node will try to discover their peers using the Kubernetes API and attempt to join them. 
The plugin requires the following access to Kubernetes resources:
* get access to the endpoints resources
* create access to the events resources.
Specify a role, role-binding, and service account to configure this access.
`kubectl apply -f rbac.yml`

3. **Service**
The Stateful Set definition can reference a Service which gives the Pods of the Stateful Set their network identity. This is required by RabbitMQ for clustering, and as mentioned in the Kubernetes documentation, has to be created before the Stateful Set. 
RabbitMQ uses port 4369 for node discovery and port 25672 for inter-node communication. Since this Service is used internally and does not need to be exposed, we create a Headless Service. 
`kubectl apply -f services.yml`

4. **Stateful Set**
RabbitMQ requires using a K8s Stateful Set to deploy a RabbitMQ cluster to Kubernetes. The Stateful Set ensures that the RabbitMQ nodes are deployed in order, one at a time. This avoids running into a potential peer discovery race condition when deploying a multi-node RabbitMQ cluster.
Some other reasons for using a Stateful Set instead of a Deployment are: sticky identity, simple network identifiers, stable persistent storage and the ability to perform ordered rolling upgrades. The Stateful Set definition file is packed with detail such as mounting configuration, mounting credentials, opening ports, etc. 
`kubectl apply -f statefulset`

5. **Persistent Volume**
In order for RabbitMQ nodes to retain data between Pod restarts, node's data directory must use durable storage. A Persistent Volume must be attached to each RabbitMQ Pod.
If a transient volume is used to back a RabbitMQ node, the node will lose its identity and all of its local data in case of a restart. This includes both schema and durable queue data.
In our statefulset.yaml example, we create a Persistent Volume Claim to provision a Persistent Volume.
  The Persistent Volume is mounted at /var/lib/rabbitmq/mnesia. This path is used for a RABBITMQ_MNESIA_BASE location: the base directory for all persistent data of a node. A description of default file paths for RabbitMQ can be found in the RabbitMQ documentation. Node's data directory base can be changed using the RABBITMQ_MNESIA_BASE variable if needed. Make sure to mount a Persistent Volume at the updated path.

6. **Node Configuration**
The recommended way to configure a R-MQ node is to use the configuration file, which can be expressed as Config-Map and mounted as a Volume onto the R-MQ pods.
`kubectl apply -f configmap.yaml`

#### Ports
Protocols supported by RabbitMQ are all TCP-based and require the protocol ports to be opened on the RabbitMQ nodes. Depending on the plugins that are enabled on a node, the list of required ports can vary. The example enabled_plugins file mentioned above enables a few plugins: rabbitmq_peer_discovery_k8s (mandatory), rabbitmq_management and rabbitmq_prometheus. Therefore, the service must open several ports relevant for the core server and the enabled plugins:
* *5672*: used by AMQP 0-9-1 and AMQP 1.0 clients
* *15672*: management UI and HTTP API)
* *15692*: Prometheus scraping endpoint)

#### Plugins 
Some plugins are essential when running RabbitMQ on Kubernetes, ex: the Kubernetes-specific peer discovery implementation. For exmaple, The rabbitmq_peer_discovery_k8s plugin is required to deploy RabbitMQ on Kubernetes. It is quite common to also enable rabbitmq_management plugin in order to get a browser-based management UI and an HTTP API, and rabbitmq_Prometheus for monitoring.
The recommend way to establish the plugin is by mounting the plugins file, enabled_plugins, to the node configuration directory, /etc/rabbitmq. A Config Map can be used to express the value of the enabled_plugins file. It can then be mounted as a Volume onto each RabbitMQ container in the Stateful Set definition.


### Requirement for test Java code/program:

The sample Sender-Receiver Java program uses AMQP 0-9-1 protocol, which is an open, general-purpose protocol for messaging and uses Java client provided by RabbitMQ
Here's the link for these libraries:
1. [AMQP Client Library](https://repo1.maven.org/maven2/com/rabbitmq/amqp-client/5.7.1/amqp-client-5.7.1.jar)
2. Dependencies for client libraries includes:
[SLF4J API](https://repo1.maven.org/maven2/org/slf4j/slf4j-api/1.7.26/slf4j-api-1.7.26.jar)
[SLF4J Simple](https://repo1.maven.org/maven2/org/slf4j/slf4j-simple/1.7.26/slf4j-simple-1.7.26.jar)

The client library and it's dependency will be in the same working directory. 


## Sample Java Program (Sender-Receiver)

1. SendK8s.java: simple java program to send "Hello World!" to queue "hello".
2. RecvK8s.java: simple java program to read the message from queue "hello".

Please note:
* host value (under setHost) will be replaced with the node ip (in case of NodePort), we can get the value from "minikube ip".
* port value (under setPort) is set in the services.yml (under name: amqp -> nodePort)

##### Compile the java codes with RabbitMQ java client on the classpath:

`javac -cp amqp-client-5.7.1.jar SendK8s.java RecvK8s.java`

##### Run the java program by adding the rabbitmq client and its dependency on the classpath


`java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar RecvK8s`
`java -cp .:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar SendK8s`


Alternatively, we can set the env variable with rabbitmq client and it's dependency 

`export CP=.:amqp-client-5.7.1.jar:slf4j-api-1.7.26.jar:slf4j-simple-1.7.26.jar`
`java -cp $CP RecvK8s`
`java -cp $CP SendK8s`







