Call Flows
----------

We have had many questions about how Clearwater handles the failure of individual node instances without disruption to the services it supports. Usually, these questions revolve around how Clearwater can process SIP requests statelessly, given that some long-lived state (such as registration state) is essential to the operation of an IMS core. The material presented here aims to provide answers to those questions, and to provide some clarity around what happens when Clearwater node instances fail - as they inevitably will from time to time.

### SIP State Definitions

The SIP protocol defines two types of state: transaction state and dialog state. A SIP transaction is initiated by a SIP request and is terminated by a final response. For example, a SIP transaction may begin with an INVITE request, and the transaction is terminated by a final response such as 200 OK or 404 Not Found (although technically in the latter case the transaction also includes the subsequent ACK). SIP transactions are therefore relatively short-lived. A SIP dialog is initiated by certain types of SIP request such as INVITE or SUBSCRIBE. A SIP dialog continues until terminated by certain types of SIP request such as BYE. SIP dialogs generally persist for substantial periods of time, e.g. for the duration of voice call. In general, Clearwater nodes store transaction state locally. The loss of a node instance therefore means a loss of transaction state, and therefore the failure of any outstanding transactions in progress on that node instance. This is consistent with the fault tolerance techniques that are commonly used in call processing systems. Calls are generally not protected against the failure of any component or sub-system until they have progressed to an established (i.e. connected) state. No state longer-lived than a transaction is stored by Sprout or Dime nodes, and so loss of Sprout and Dime nodes never results in the loss of dialogs. Longer-lived state is only stored by Vellum nodes, and then it is stored in distributed, cloud optimized, fault tolerant databases (like Cassandra). As a result the loss of an individual Vellum node instance never results in the loss of dialogs either.

### Clustering Architecture

Before getting into the details of call flows, we need to describe the way that Clearwater uses clustering and stores the necessary state. The diagram below illustrates the clustering and state storage architecture of Clearwater. Note that we are assuming the use of an external P-CSCF and I-BCF (so the bono nodes are not present) and we are assuming that Rf billing is not configured (and so we don’t discuss flows to the CDF). We also assume in the flows below that Clearwater is configured to optimise out I-CSCF lookups of the HSS for performance reasons.

![Clearwater Clustering](https://www.projectclearwater.org/wp-content/uploads/2017/05/Clearwater-Clustering-May-2017.png)

The components are as follows:

####Home Subscriber Service
The Home Subscriber Service (HSS) is the master store for all subscriber data. It exposes a Cx/Diameter interface for retrieving and updating data.

####Dime
Dime is Clearwater’s Diameter gateway. It is a
cluster of 2 or more nodes with each node running:

* Homestead: Clearwater's subscriber data cache. It exposes an HTTP interface for Sprout to retrieve subscriber data. The subscriber data is retrieved from the HSS via the Cx/Diameter interface and cached by Homestead in the deployment’s distributed data
stores (hosted on Vellum).
* Ralf: Collects billing events from other Clearwater components and implements the Rf billing interface to the CDF. As above, we are assuming that Rf billing is not used in this sample deployment and so this is not discussed further here.

####Sprout
Sprout is Clearwater's SIP router. It receives requests from the P-CSCF and unsolicited requests from ASs, queries subscriber data from Dime (Homestead), routes via the ISC (SIP) interface to ASs, performs ENUM queries, acts as a registrar, and routes SIP requests towards registered endpoints. It is a cluster of 2 or more nodes and each node independently maintains a store of active transactions. Sprout does not retain any longer-lived state, with Sprout instead using Vellum to store long-lived registration state and to run timers.

####Vellum
Vellum is Clearwater’s store for long-lived data, storing:

* Registration state (from Sprout, in a memcached cluster)
* Subscriber profiles and authentication credentials (retrieved from the HSS by Homestead, in a Cassandra cluster).

It also runs timers for Sprout and Dime, using a distributed Chronos timer store. For simplicity, Vellum is not shown in the sample flows below as all flows are purely internal between Vellum and other Clearwater nodes. Where the flows read e.g. “Homestead retrieve subscriber profile from Cache”, in reality there is a roundtrip from Homestead to Vellum to retrieve that data.

####Other
* The ENUM server simply responds to ENUM (DNS) queries from Sprout to map telephone numbers to SIP URIs.
* The Application Server (AS) provides additional services, such as local and national dialing plans and call services. It receives requests from Sprout and either handles them itself or passes them back to Sprout for further processing. It may also send unsolicited requests - these go to Sprout.

### Communication and State

Sprout, Dime and Vellum all consist of clusters of nodes. Each node has an IP address (and possibly a domain name). Additionally, each cluster has a cluster domain name, which resolves to all the IP addresses in the cluster. The Dime and Vellum cluster domain names are only used within Clearwater itself; e.g. Sprout always uses the Dime and Vellum cluster domain names to interface with Dime and Vellum rather than addressing individual nodes. The Sprout cluster domain name is used by the P-CSCF and AS. The P-CSCF and AS must support RFC 3263, which covers DNS round-robin behavior. If a transaction the P-CSCF or AS sends towards the Sprout cluster fails because the chosen Sprout node does not respond, the P-CSCF or AS must retry the transaction to a different Sprout node.

####Sprout
Sprout nodes are transaction-stateful but not dialog-stateful. If a request flows through a specific Sprout, the corresponding response(s) must also flow through that Sprout. Each Sprout node includes its own IP address in Via headers to ensure that responses are routed back via the same Sprout node as the request. The Sprout cluster stays in the dialog's signaling path by record-routing itself using the cluster domain name. The cluster domain name resolves to all of the Sprout IP addresses, so there is no need for a subsequent in-dialog transaction to be processed by the same Sprout instance as the original dialog-initiating transaction.

####Dime (Homestead)
Homestead processes HTTP requests, not SIP requests. It is technically HTTP-transaction-stateful, but since it always responds to HTTP requests itself, these transactions are very short-lived (generally low tens of milliseconds).

### **Mainline Flows**

Before diving into the detail of what happens on node failure, consider the normal flows when

*   registering
*   processing dialogs, including initiation, in-dialog requests and termination
*   processing out-of-dialog requests such as MESSAGE.

####Registration
![register](https://www.projectclearwater.org/wp-content/uploads/2014/02/register.png)

####Dialog Initiation (on-net)
![dialog-initiation](https://www.projectclearwater.org/wp-content/uploads/2014/02/dialog-initiation.png)
![dialog-initiation2](https://www.projectclearwater.org/wp-content/uploads/2014/02/dialog-initiation2.png)

####In-Dialog Request
![in-dialog](https://www.projectclearwater.org/wp-content/uploads/2014/02/in-dialog.png)

####Dialog Termination
![dialog-termination](https://www.projectclearwater.org/wp-content/uploads/2014/02/dialog-termination.png)

####Out-Of-Dialog Request
![out-of-dialog](https://www.projectclearwater.org/wp-content/uploads/2014/02/out-of-dialog.png)

### **Behavior on Node Failure**

####Sprout
If a sprout node fails before receiving a request from the P-CSCF, the request fails and the P-CSCF retries.
![sprout-initial-request](https://www.projectclearwater.org/wp-content/uploads/2014/02/sprout-initial-request.png)
If a sprout node fails after a dialog is established, again the P-CSCF or UE retries - the Route header specifies the sprout cluster, so the P-CSCF applies round-robin DNS processing to them.
![sprout-dialog-initiation](https://www.projectclearwater.org/wp-content/uploads/2014/02/sprout-dialog-initiation.png)
If a sprout node fails while a transaction is in progress, the transaction fails. Either the UE will retry automatically or it will display an error to the user, who should retry.
![sprout-pending-transaction](https://www.projectclearwater.org/wp-content/uploads/2014/02/sprout-pending-transaction.png) This transaction failure scenario occurs even if the only operation the sprout node has performed is sending the 100 Trying response to an INVITE, as from that point on the P-CSCF will not retry the transaction. All of the above apply equally when the request is an unsolicited request sent by an AS, rather than a request from a P-CSCF. Sprout stores all Registration state in Vellum so it is not tied to any specific Sprout node; If a user registers via one Sprout node and that Sprout node then fails, subsequent messages for that user can still be routed by any other Sprout node. Note that Sprout also has interfaces to the ENUM server, Dime and Vellum. In all cases, Sprout issues requests and waits for responses before continuing processing a SIP request. As a result, a Sprout failure in either case is equivalent to the Sprout node failing while processing the SIP request, resulting in transaction failure back to the UE.

####Dime (Homestead)
Homestead's HTTP interface is simple. If one homestead instance does not respond to sprout, sprout tries a different one.
![homestead_fail](https://www.projectclearwater.org/wp-content/uploads/2013/10/homestead_fail.png)

####Vellum
Vellum is always addressed by its cluster name and all of its data is stored in distributed databases with replicas of data on multiple nodes. If a Vellum node fails then no data is lost, and Sprout / Dime can continue to access the data via the Vellum cluster domain name. If a Vellum node fails whilst a request from Sprout or Dime is outstanding then this will typically result in failure of the request that the Sprout / Dime node was processing (with a return code indicating that it should be retried). When the request is retried an alternative Vellum node will be used instead.

