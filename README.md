# VanFlow Specification

VanFlow is a specification for the issuance of events from components of the
implementation of a Virtual Application Network (VAN) for the purpose of
observing the operation of the VAN.  VanFlow is similar in concept to NetFlow,
but differs in that VanFlow provides data at a higher level of abstraction and
touches higher level concepts like processes and network topology.

## Transport

The VAN implementation that VanFlow is built for is the Skupper implementation.
Since Skupper is built on the foundation of an AMQP network, the VanFlow
content is transferred using AMQP messages and multicast addressing.

Reference the [AMQP 1.0 Specification][amqp].

VanFlow messages are AMQP-formatted messages with the following characteristics

- The **Properties.to** field contains the destination address of the message
- The **Properties.subject** field contains the operation-code for the message.
  The operation codes are BEACON, RECORD, and FLUSH
- The **ApplicationProperties** section contains operation-code-specific
  arguments
- The **Body** section is an **AMQPValue** section containing
  operation-code-specific contents

The following addresses are used when transferring VanFlow messages:

- `mc/sfe.all` - BEACON messages are sent via this address (multicast
  forwarding.) This is the only well-known address an event collector requires
  to participate in the VanFlow protocol. BEACON messages should contain the
  addresses to use in order to communicate with that event source.
- `sfe.<source-identity>` - FLUSH messages are sent to this address.  The source-identity
  is the identity of the event source to which the message is being sent
  (anycast forwarding)
- `mc/sfe.<source-identity>` - RECORD and HEARTBEAT messages are sent via this address from the
  source that has the included identity (multicast forwarding)
- `mc/sfe.<source-identity>.flows` - event sources that publish FLOW records
  will send RECORD messages containing FLOW records via this suffixed address
  instead of the default address. This way event collectors can opt-in to
  receiving the high volume events associated with FLOWS.


## Message Types

### BEACON

The BEACON message is transmitted periodically by every VanFlow event source.
Event sources include ROUTERs, CONTROLLERs, and COLLECTORs.  All BEACON
messages are sent using the mc/sfe.all multicast address.

The purpose of the BEACON message is to allow event collectors to discover
sources of events.  Once a collector has discovered the existence of an event
source, it may optionally subscribe to receive VanFlow record events from that
source.

| Field    | Type    | Value or Description    |
| -------- | ------- | -------- |
| Properties.To | str8-utf8 | mc/sfe.all
| Properties.Subject | str8-utf8, sym8 | BEACON
| ApplicationProperties.v | smalluint | 1 (Protocol Version)
| ApplicationProperties.sourceType | str8-utf8 | The record type for the event source (e.g. ROUTER, CONTROLLER, COLLECTOR)
| ApplicationProperties.address | str8-utf8 | The address on which event records are sent from this source. By convention this is `mc/sfe.<source-identity>`.
| ApplicationProperties.direct | str8-utf8 | The address to which messages can be sent directly to this source. By convention this is `sfe.<source-identity>`.
| ApplicationProperties.id | str8-utf8 | The event source’s record identifier.

### HEARTBEAT

Like the BEACON message, the HEARTBEAT message is also sent periodically by
each event source.  Unlike the BEACON message, the HEARTBEAT message is sent to
the source-specific multicast address so that it is only received by collectors
that have explicitly subscribed to events from that source.

| Field    | Type    | Value or Description    |
| -------- | ------- | -------- |
| Properties.To | str8-utf8 | mc/sfe.source-identity
| Properties.Subject | str8-utf8, sym8 | HEARTBEAT
| ApplicationProperties.v | smalluint | 1 (Protocol Version)
| ApplicationProperties.id | str8-utf8 | The event source’s record identifier.
| ApplicationProperties.now | ulong | The event source’s current time.

### RECORD

The RECORD message is transmitted by event sources.  The message carries a set
of incremental record updates that can be collected and aggregated by
collectors.  Updates are incremental in that for a given flow record, the same
attribute value shall not be sent multiple times.  Attributes are re-sent only
if their value has changed since the last update.  The only attributes that are
included in all of the updates are the RECORD_TYPE and IDENTITY attributes.

| Field    | Type    | Value or Description    |
| -------- | ------- | -------- |
| Properties.To | str8-utf8 | The address to which the RECORD message is sent. Usually mc/sfe.source-identity.
| Properties.Subject | str8-utf8, sym8 | RECORD
| Body.AMQP_Value | list32 | List of map8/map32 where the key is the Attribute Code and Value is the Attribute Value.

### FLUSH

The FLUSH message is sent by a collector to a specific event source.  When the
event source receives the FLUSH request, it responds by sending non-incremental
updates for all currently active records (see the Lifecycle section below).  A
non-incremental update includes all attributes of a record.  FLUSH is used when
a new collector comes online and discovers (via a BEACON message) a new event
source.  It allows the collector to receive a baseline of all complete active
records from the source.

| Field    | Type    | Value or Description    |
| -------- | ------- | -------- |
| Properties.To | str8-utf8 | Address of the event source as found in the BEACON.ApplicationProperties.direct field
| Properties.Subject | str8-utf8, sym8 | FLUSH

## Record Codepoints

Record codepoints are numeric values that appear in RECORD messages to
represent record types and record attribute types.

### Record Types

| Code    | Record Type | Description    |
| ------- | ----------- | -------- |
| 0 | SITE | A VAN site
| 1 | ROUTER | A VAN router
| 2 | LINK | A Link between a pair of VAN routers
| 3 | CONTROLLER | A VAN controller, usually associated with a site
| 4 | LISTENER | An incoming proxy for VAN communications on a router
| 5 | CONNECTOR | An outgoing proxy for VAN communications on a router
| 6 | FLOW | Details of a specific protocol exchange between processes in a VAN
| 7 | PROCESS | A process/pod that participates in VAN communication
| 8 | IMAGE | The software image that is run within a process
| 9 | INGRESS | An ingress into the VAN from the outside
| 10 | EGRESS | An egress through which processes in the VAN can communicate with services outside the VAN
| 11 | COLLECTOR | A VanFlow event collection component
| 12 | PROCESS_GROUP | A grouping of processes
| 13 | HOST | Host (or Kubernetes Node) on which a process runs
| 14 | LOG | A log message issued by the router.

### Record Attributes

| Code | Attribute | Data Type | Description |
| ---- | --------- | --------- | ----------- |
| 0 | recordType | uint | Record type code
| 1 | identity | record-id | This record’s identity
| 2 | parent | record-id | Parent record’s identity
| 3 | startTime | timestamp | Timestamp of record creation
| 4 | endTime | timestamp | Timestamp of record deletion
| 5 | counterflow | record-id | Identity of opposite-direction flow
| 6 | peer | record-id | Identity of peer record
| 7 | process | record-id | Identity of associated process record
| 8 | sibOrdinal | uint | This record’s rank among siblings
| 9 | location | string | Site location
| 10 | provider | string | Site provider name
| 11 | platform | string | Site platform
| 12 | namespace | string | Site/router namespace
| 13 | mode | string | Operating mode of the object
| 14 | sourceHost | string | Hostname of protocol source
| 15 | destHost | string | Hostname of protocol destination
| 16 | protocol | string | Application or network protocol
| 17 | sourcePort | string | Port of protocol source
| 18 | destPort | string | Port of protocol destination
| 19 | address | string | VAN address of protocol interconnect
| 20 | imageName | string | Name of container image
| 21 | imageVersion | string | Version of container image
| 22 | hostName | string | Hostname for this record
| 23 | octets | uint | Number of octets transferred for a flow
| 24 | latency | uint | Latency of a protocol exchange in uSec
| 25 | transitLatency | uint | Latency across a network in uSec
| 26 | backlog | uint | Number of outstanding protocol units being handled by a process
| 27 | method | string | Protocol method
| 28 | result | string | Protocol result
| 29 | reason | string | Protocol error reason
| 30 | name | string | Name of a record
| 31 | trace | string | List of routers through which a flow traveled
| 32 | buildVersion | string | Build version for this record
| 33 | linkCost | uint | Link cost associated with an inter-router link
| 34 | direction | string | Incoming or Outgoing - attribute of an inter-router link
| 35 | octetRate | uint | Deprecated. Rate of change of the OCTETS attribute as a five-second average
| 36 | octetsOut | uint | Octets outbound (from the router) on a flow.
| 37 | octetsUnacked | uint | Octets currently unacknowledged (in-flight).
| 38 | windowClosures | uint | Number of times a flow-control window closed and traffic flow stalled on a flow.
| 39 | windowSize | uint | The size in octets of a flow’s flow-control window.  This is the number of octets of payload that will be accepted by the router before applying back-pressure on the sender.
| 40 | flowCountL4 | uint | Deprecated. The number of Layer-4 flows associated with an endpoint.
| 41 | flowCountL7 | uint | Deprecated. The number of Layer-7 flows associated with a Layer-4 flow or an endpoint.
| 42 | flowRateL4 | uint | Deprecated. The rate at which Layer-4 flows are created in the context of an endpoint.
| 43 | flowRateL7 | uint | Deprecated. The rate at which Layer-7 flows are created in the context of a Layer-4 flow or an endpoint.
| 44 | duration | uint | The number of microseconds that this record has been active.  This value is computed and generated at the event collector.
| 45 | image | record-id | Reference to an IMAGE record.
| 46 | group | record-id | Reference to a process group.
| 47 | streamId | uint | Stream identity for flows if relevant to the protocol.
| 48 | logSeverity | uint | Severity level associated with a log message.
| 49 | logText | string | The log message text.
| 50 | sourceFile | string | Source code filename and path. Used to locate the origin of a log event (Optional based on router log configuration - used for debugging).
| 51 | sourceLine | uint | Line number in sourceFile where log event was generated (Optional based on router log configuration - used for debugging).

## Attributes per Record Type

This section identifies the attributes that can be included in records of each record type.

It should be noted that the following two attributes are mandatory in every record and for every record update:

- recordType
- Identity

Furthermore, the following attributes are mandatory for every record but may not be included in all updates:

- startTime
- endTime

### SITE

| Attribute | Meaning in context |
| --------- | ------------------ |
| location | Location of the cloud site geographically (i.e. Boston, USA)
| provider | If appropriate, the provider of the cloud platform (i.e. AWS, Azure AKS, etc.)
| platform | Description of the cloud platform (i.e. Kubernetes/version, RHEL/version, etc.)
| namespace | If appropriate, the namespace of the cloud site, for example in a kubernetes cluster.

### ROUTER

| Attribute | Meaning in context |
| --------- | ------------------ |
| parent | Reference to the owning SITE record
| namespace | If appropriate, the namespace in which the router process is running
| mode | The router’s operating mode (i.e. interior, edge)
| imageName | The name of the image for the router process
| imageVersion | The version of the image for the router process
| hostname | The host (or pod) name for the router process
| name | The name of the router
| buildVersion | The version of the upstream code for the process

### LINK

| Attribute | Meaning in context |
| --------- | ------------------ |
| parent | Reference to the owning ROUTER record
| mode | The mode of the inter-router link (i.e. inter-router, edge)
| name | The name of the router to which this link is attached
| linkCost | The cost metric of the link
| direction | The direction of establishment for the link (incoming, outgoing)

### CONTROLLER

| Attribute | Meaning in context |
| --------- | ------------------ |
| parent | Reference to the owning SITE record
| imageName | The name of the image for the controller process
| imageVersion | The version of the image for the controller process
| hostname | The host (or pod) name for the controller process
| name | The name of the controller
| buildVersion | The version of the upstream code for the process

### LISTENER

| Attribute | Meaning in context |
| --------- | ------------------ |
| parent | Reference to the owning ROUTER record
| destHost | Destination IP/Host for the listener socket
| destPort | Port for the listener socket
| protocol | The protocol for the listener socket (i.e. TCP, HTTP, etc.)
| address | The VAN address for encapsulated socket traffic
| flowCountL4 | Deprecated. Number of layer-4 flows established through this listener
| flowRateL4 | Deprecated. The rate of layer-4 flow establishment through this listener (in flows/sec)
| flowCountL7 | Deprecated. Number of layer-7 flows through this listener
| flowRateL7 | Deprecated. The rate of layer-7 flow creation through this listener

### CONNECTOR

| Attribute | Meaning in context |
| --------- | ------------------ |
| parent | Reference to the owning ROUTER record
| destHost | Destination IP/Host for the target workload
| destPort | Destination port for the target workload
| protocol | The protocol for the target workload
| address | The VAN address for encapsulated service traffic
| flowCountL4 | Deprecated. Number of layer-4 flows established to this server
| flowRateL4 | Deprecated. The rate of layer-4 flow establishment
| flowCountL7 | Deprecated. Number of layer-7 flows to this server
| flowRateL7 | Deprecated. The rate of layer-7 flow creation to this server

### FLOW

| Attribute | Meaning in context |
| --------- | ------------------ |
| parent | Reference to the owning CONNECTOR or LISTENER record
| sourceHost | IP/Host of the source/client endpoint of this flow.  Note that this is the router/proxy host for children of CONNECTORs.
| sourcePort | Port of the source/client endpoint
| counterflow | Reference to the opposite-direction flow associated with this flow
| trace | Verticle-bar separated list of routers through which this flow traveled from source to destination
| latency | The latency in microseconds from the arrival of the first octet of client traffic to the arrival of the first octet of server traffic.  For children of CONNECTORS, this is the latency of the server process.  For children of LISTENERS, this is the latency experienced on the client side, including network-traversal (round-trip) time.
| octets | The total count of octets of payload that have traversed this flow.
| octetRate | Deprecated. The current (rolling average) rate of octets traversing the flow in octets/second
| octetsOut | The total count of octets outbound from the router for this flow.  This value should be redundant with the octets count of the counterflow.
| octetsUnacked | The current number of octets traversing this flow that have not been acknowledged by the destination.
| windowClosures | The count of times that octetsUnacked reached the windowSize, causing a stall in traffic flow for back-pressure.
| windowSize | The size of the window, in octets, used for flow control.
| reason | If there was an error or exceptional closing condition for this flow, this string describes the reason for the error
| method | For L7 flows, this is the method, or op-code of the request being sent to the server.
| result | For L7 flows, this is the result code from the server response.  For HTTP flows, this is a string containing the three digit result code number.
| process | Reference to the process record that is this flow’s non-router/proxy endpoint.
| streamId | Stream identity for http2 flows.

### PROCESS

| Attribute | Meaning in context |
| --------- | ------------------ |
| name | The process name.  In kubernetes, this is the pod name.
| imageName | The full image name for the process, including the version tag.
| image | Reference to an IMAGE record
| group | Reference to a PROCESS_GROUP record
| sourceHost | The IP address on which this process is accessed

## Record Lifecycle

### Event Source

The lifecycle of a VanFlow record is observable via the START_TIME and END_TIME
attributes.  START_TIME is the local timestamp stored at the moment the record
is created on the event source.  Every record has a start time.

END_TIME is provided in the record’s final update as the timestamp of the
moment when the record was deleted on the event source.

Any record that has a START_TIME but no END_TIME is considered “active” as it
represents a currently-existing object in the context of the event source.
Once a record is deleted and the terminal update is sent, the record is removed
and no further updates are ever sent, even in response to a FLUSH request.

### Event Collector

Within the context of an event collector, records are not necessarily deleted.
A record that has an END_TIME will remain available for viewing as an
historical record, representing what has happened in the past.

It is also possible for a record to become stale.  In other words, the record’s
lifecycle may be considered complete even if the collector does not receive an
explicit end-time for the record.  The collector may keep track of the last
time an active record was updated.  If this time becomes too far in the past,
the collector may mark the record as stale and exclude it from the results of
queries looking for active records.

The implementation of an event collector may optionally purge historical
records for the purpose of saving memory and storage space.

## Identifiers and References

Every VanFlow record has a unique identifier.  Records may contain explicit
references to other records by storing the other record’s identifier in a
reference attribute (PARENT, COUNTERFLOW, etc).

Record identifiers are arbitrary strings.

SITE record identity must be stable throughout the lifecycle of the VAN
network.  As such, it must not contain any text that might change if the
emitting source is re-started.

## Hierarchy and Peer Linkage

VanFlow records are arranged in a hierarchical and connected structure using
identity reference attributes.  The hierarchy is established via the PARENT
attribute.  The following diagram illustrates the record-type hierarchy:

![VanFlow Record Peer Linkage](/VanFlowPeerRelations.png)

In the above diagram, any linkage that is unlabelled should be assumed to be
PARENT.  Note that FLOW records can be children of either CONNECTOR, LISTENER,
or other FLOW records.  A FLOW within a FLOW represents a layer-7 protocol flow
encapsulated within a layer-4 protocol flow (e.g. HTTP within TCP, where TCP is
the parent of HTTP).

### FLOW Linkage - Layer 4 Protocol

The following diagram illustrates the record structure for a layer-4 VAN
interconnect.  The interaction between client and server is represented by a
pair of flows that are inter-linked via their COUNTERFLOW references.  Each
flow is a member of the hierarchy from which it was configured, via a LISTENER
for the client-side and CONNECTOR for the server-side.  Each flow is also
linked via a PROCESS reference to a PROCESS record.

![VanFlow Layer 4 Flow Relations](/VanFlowL4Flows.png)

### FLOW Linkage - Layer 7 Protocol

The diagram for a layer-7 interchange is similar to the layer-4 picture except
that there is a two-deep hierarchy of FLOW records.  Note that the lowest level
flows (for the layer-7 protocol) have the COUNTERFLOW linkages and that the
higher level flows (for the layer-4 protocol) have the PROCESS linkages.

![VanFlow Layer 7 Flow Relations](/VanFlowL7Flows.png)

### Collector-Inferred Linkage

Some of the links that form associations between records cannot be supplied by
the original event source and must be added later by the collector.

FLOW to PROCESS linkages are formed by correlating the IP addresses in
SOURCE_HOST and DESTINATION_HOST to the IP addresses (and possibly ports) of
the PROCESS records.

COUNTERFLOW linkages are provided in only one direction by the event sources.
The connector-side flow contains a counterflow link to the listener-side flow.
The collector should add in the opposite-direction counterflow link for the
listener-side flow record.  This causes the flow records to mutually reference
each other as counterflows.

[amqp]: http://docs.oasis-open.org/amqp/core/v1.0/os/amqp-core-complete-v1.0-os.pdf
