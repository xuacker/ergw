CLUSTER
=======

Theories:

 1. A cluster of ergw nodes survives a single GTP-C node failure without
	loosing user sessions.
 2. A cluster of ergw nodes survives a multiple GTP-C node failures without
	loosing user sessions.
 3. A cluster of ergw nodes survives a single GTP-U node failure without
	loosing user sessions.
 4. A cluster of ergw nodes survives a multiple GTP-U node failures without
	loosing user sessions.

Abstract
--------

Implementing all four failure modes for Gi/SGi traffic termination without PCEF
functionality is doable with modest complexity. When adding PCEF functionality
into the solution the required QoS and limits enforcing features add a high
level of complexity that needs further investigation.

Modes of redundancy
===================

We will define and investigate the following redundancy and error recovery
strategies and their applicability to GTP-C and GTP-U in the next chapters.

From a high level perspective all solutions can be grouped by whether they
maintain tunnel states during a failure or loose tunnel states (and possibly
recover them from other sources).

The following setups will be investigated:

  * single (same) IP on multiple hosts
  * load balancer/traffic director switching to multiple back-end hosts
  * IP take-over with a OS level cluster management

For setups that loose tunnel states the possible restoration modes will be
investigated and for setups that maintain tunnel states in a cluster setup,
the behavior during single node failure will be investigated in a separate
chapter.

GTP endpoint redundancy implementations
=======================================

A implementation scenarios boil to three underlying architectures:

 * multiple paths with load sharing
 * take over of failed functionality by a standby node
 * orderly failure notification and reestablishment by the UE

NOTE: GTP endpoints a uniquely identified by endpoint IP and Tunnel Endpoint
      Identifier. When we talk in the following sections about a node change,
	  this always implies that the other node has a different IP.

multiple paths with load sharing
--------------------------------

### single (same) IP on multiple hosts.

The same IP is assigned to and active on multiple hosts. Those hosts can not
share the same L2 network. Traffic steering is archived through routing.

All nodes will active and only the routing costs decide which node is actually
use. Routing is a per-packet decision, so traffic paths can change with every
packet.

From the outside this will appear as if the IP is multi homed and can thus be
reached through multiple paths. The path costs will determine which path is
taken.

Announcing the routes through a routing protocol (e.g. OSPF, BGP, ...) will
take care of fail-over to another host should one of the routes or hosts fail.

Advantages:

  * geo redundancy comes for free

Disadvantages:

  * tunnel state need to be replicated to all nodes

#### Application to GTP-C for a P-GW/GGSN

The endpoint will appear as a single hosts (even though there are in reality
multiple hosts). Therefore the following information need to be globally
synced or available:

  * recovery counter
  * session state for a given TEID

#### Application to GTP-U for a P-GW/GGSN

The endpoint will appear as a single hosts (even though there are in reality
multiple hosts). Therefore the following information need to be globally
synced or available:

  * recovery counter
  * session state for a given TEID
  * IP address assigned to a client for a give TEID needs to be route-able
  * accounting information need to be aggregated

### load balancer/traffic director switching to multiple back-end hosts

A flow aware (or rather GTP Tunnel Id aware) UDP load balancer (LB) or
traffic director (TD) is distributing the tunnels to multiple nodes.

Disadvantages:

  * load balancer/traffic director becomes a single point of failure
  * no geo redundancy for established tunnels

The actual setup of the GTP-C nodes could be independent nodes or any of the
cluster modes (see discussion there).

#### Application to GTP-C for a P-GW/GGSN

The LB/TD is the anchor point for the tunnel. Request can distributed to worker
with a number of methods. Round-robin or similar distribution assignment makes
this a load sharing setup. Static assignment based in TEID would make this a
take over of failed functionality by a standby node setup.

#### Application to GTP-U for a P-GW/GGSN

Same as GTP-C for this setup

take over of failed functionality by a standby node
---------------------------------------------------

### IP take-over with a OS level cluster management

Multiple solutions with roughly the same demand on the application behavior
exists:

  * A OS level cluster solution (e.g. heartbeat, Pacemaker, ...) monitors host
    and service states. On failure the endpoint it is move to another host and
	a new erGW instance for the service IP is activated on that host.

  * based on virtualization infrastructure, something monitors host and service
    health, terminates VM when the service fails and brings up a new VM instance
	to take the place of the old VM

  * the restart of failed GTP node (without restart of the underlying OS) can
     be seen as a special version of this case (IP and functionality is taken
	 over by the same node)

Common to all this solutions is that the GGSN/P-GW service on a given service
IP is terminated, it's local state is lost and new, fresh instance takes it
place. If desirable, state can be restored from a GTP-C cluster setup.

#### Application to GTP-C for a P-GW/GGSN

The failed nodes IP address is activated on a new host. For the erGW two options
exists:

  * a erGW is already listen to that IP (freebind) and has synchronized state
    for all GTP tunnels. It can immediately process requests for the GTP-C
    contexts and needs to contact the GTP-U nodes for the data plane bearers.

  * a erGW instance is started to handle the IP, it retrieves the GTP-C context
    states before it can process requests for the GTP-C contexts and contact
	the GTP-U nodes for the data plane bearers. This also includes the case
	where a simple erGW restart is performed without moving the IP to another
	system.

#### Application to GTP-U for a P-GW/GGSN

The failed nodes IP address is activated on a new host and in the case of
GGSN/P-GW the client IP pool that was previously assigned to the failed node is
routed to the new node (this can either happen through a routing protocol or
through a simple take-over of the routing IP address).
Incoming requests for unknown TEIDs are forwarded to GTP-C management instance
that either recreates the bearer in the GTP-U node or triggers the proper
error responses.

### load balancer/traffic director switching to multiple back-end hosts

The setup is identical to the LB/TD switching but the distribution uses
a sticky selection that switches processing and forwarding nodes only in
case of node failure.

### take-over of sessions for a new endpoint

A GTP-C cluster with distributed session state could attempt to redirect all
tunnels from a failed node another node.

For GTP-C P-GW/GGSN this needs a mechanism to tell the SGSN/S-GW that the GTP-C
and GTP-U GW node should be changed.

Unfortunately, no expressed mechanism for moving the GGSN/P-GW side of a GTP
endpoint that is initiated by the GGSN/P-GW exists.

orderly failure notification and reestablishment by the UE
----------------------------------------------------------

### explicit context, bearer or session deletion from P-GW

For Gn/Gp the `PDP Context Deactivation Initiated by GGSN procedure` can be used
to delete contexts from the GGSN, for S5/S8 the `PGW initiated bearer
deactivation procedures (using S4)` can be used.

### 3GPP TS 23.007, Section 10, restoration procedures for GGSN

3GPP TS 23.007 restoration procedures for GGSN define the correct cleanup
of all open PDP contexts. In the end the UE has to reestablish the context.

It would be possible to use the `Network-Requested PDP Context Activation
procedure` to establish a context to be recreated from the GGSN side. That
procedure is indented for PDP context with static assigned IP and is unclear
if that would work in a error recovery case.

### 3GPP TS 23.007, Section 17, restoration procedures for P-GW

This is mostly identical to the GGSN procedures.

An additional optional feature to the partial failure handling. Here the
P-GW can indicate that the state of a subset of connections was lost and
that all bearers belonging to a Connection Set have failed and should
be considered dead by the peer nodes.

A use case would be the failure of a GTP-U node that carries as subset of
the total connections.

GTP-U termination and PCEF redudancy
====================================

Teh GTP-U to Gi/SGi termination requires in most cases a policy control
enforcement function (PCEF). If the GTP-U are redundant then the PCEF
needs to be redundant as well.

For PCEF redundancy the accounting, QoS and traffic limit enforcement are
the key points to support.

Accounting means the counting, summarizing and reporting of the user traffic
in bytes and packets in up and down stream direction.

QoS means the enforcement of bandwidth limits.

Traffic limit enforcement means the counting and summarizing of the user
traffic in bytes and packets in up and down stream direction and the
application of rules once a certain threshold has been crossed.

multiple paths with load sharing
--------------------------------

### Accounting

Accounting is only reported in defined intervals and at the end of a
session. Therefore aggregation of counters from different nodes is
straight forward (gather all values and add).

### QoS

Open Questions:

  * How to calculate used bandwidth for a client across multiple nodes

### Limits Enforcement

  * How to aggregate counters from different nodes so that a give
    trigger values can be detected with a give granularity?

	Every packet passing through a GTP-U might hit multiple PCC rules
	that cause a traffic counter to hit a trigger value. Implementation
	choices:

	  * per GTP-U node, per PCC rules counters that are aggregated and
	    checked for trigger condition at regular intervals. The length
		of the aggregation intervals determines the achievable granularity
		of the trigger (e.g for a UE with 100mbit/s throughput and
		a 10 second aggregation intervals, a trigger condition could
		be overrun by as much as 100mbit/s * 10s = 1000mbit (100MByte) before
		a trigger action is taken.

		Also, the aggregation interval length has a impact on the scalability of
		the system.

	  * streaming of PCC counter actions

	    Every hit of PCC rule with a counter is streamed to aggregation system.
	    The traffic load generate by this is smaller that the full payload
		stream since only the PCC rule id and counter change needs to be
		communicated. The maximum overrun between hitting a trigger condition
		and taking the appropriate action is only determined by the time taken
		to transfer and process the counter event.

take over of failed functionality by a standby node
---------------------------------------------------

### Accounting

The accounting state is distributed in regular intervals to the cluster.
On failure the latest state is used. Traffic between the last distributed
state and the failure is lost in the accounting.

### QoS

QoS limits are calculated from scratch on the new node. This might to
a short spike in permitted bandwidth usage since the history is not known.

### Limits Enforcement

Like the accounting state, the traffic limits state is distributed in regular
intervals to the cluster. On failure the latest state is used. Traffic between
the last distributed state and the failure is not considered for the limits.

orderly failure notification and reestablishment by the UE
----------------------------------------------------------

### Accounting

The charging system (offline or online) will detect the GW failure and use
the last known values to close the session.

### QoS

n/a

### Limits Enforcement

The charging system (offline or online) will detect the GW failure and use
the last known values to close the session and any granted buckets.
