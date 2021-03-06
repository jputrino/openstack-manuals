==========================
High availability concepts
==========================

High availability systems seek to minimize two things:

**System downtime**
  Occurs when a user-facing service is unavailable
  beyond a specified maximum amount of time.

**Data loss**
  Accidental deletion or destruction of data.

Most high availability systems guarantee protection against system downtime
and data loss only in the event of a single failure.
However, they are also expected to protect against cascading failures,
where a single failure deteriorates into a series of consequential failures.
Many service providers guarantee :term:`Service Level Agreement (SLA)`
including uptime percentage of computing service, which is calculated based
on the available time and system downtime excluding planned outage time.

Redundancy and failover
~~~~~~~~~~~~~~~~~~~~~~~

High availability is implemented with redundant hardware
running redundant instances of each service.
If one piece of hardware running one instance of a service fails,
the system can then failover to use another instance of a service
that is running on hardware that did not fail.

A crucial aspect of high availability
is the elimination of single points of failure (SPOFs).
A SPOF is an individual piece of equipment or software
that causes system downtime or data loss if it fails.
In order to eliminate SPOFs, check that mechanisms exist for redundancy of:

- Network components, such as switches and routers

- Applications and automatic service migration

- Storage components

- Facility services such as power, air conditioning, and fire protection

In the event that a component fails and a back-up system must take on
its load, most high availability systems will replace the failed
component as quickly as possible to maintain necessary redundancy. This
way time spent in a degraded protection state is minimized.

Most high availability systems fail in the event of multiple
independent (non-consequential) failures. In this case, most
implementations favor protecting data over maintaining availability.

High availability systems typically achieve an uptime percentage of
99.99% or more, which roughly equates to less than an hour of
cumulative downtime per year. In order to achieve this, high
availability systems should keep recovery times after a failure to
about one to two minutes, sometimes significantly less.

OpenStack currently meets such availability requirements for its own
infrastructure services, meaning that an uptime of 99.99% is feasible
for the OpenStack infrastructure proper. However, OpenStack does not
guarantee 99.99% availability for individual guest instances.

This document discusses some common methods of implementing highly
available systems, with an emphasis on the core OpenStack services and
other open source services that are closely aligned with OpenStack.
These methods are by no means the only ways to do it;
you may supplement these services with commercial hardware and software
that provides additional features and functionality.
You also need to address high availability concerns
for any applications software that you run on your OpenStack environment.
The important thing is to make sure that your services are redundant
and available; how you achieve that is up to you.

Stateless vs. stateful services
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Preventing single points of failure can depend on whether or not a
service is stateless.

Stateless service
  A service that provides a response after your request
  and then requires no further attention.
  To make a stateless service highly available,
  you need to provide redundant instances and load balance them.
  OpenStack services that are stateless include ``nova-api``,
  ``nova-conductor``, ``glance-api``, ``keystone-api``,
  ``neutron-api`` and ``nova-scheduler``.

Stateful service
  A service where subsequent requests to the service
  depend on the results of the first request.
  Stateful services are more difficult to manage because a single
  action typically involves more than one request, so simply providing
  additional instances and load balancing does not solve the problem.
  For example, if the horizon user interface reset itself every time
  you went to a new page, it would not be very useful.
  OpenStack services that are stateful include the OpenStack database
  and message queue.
  Making stateful services highly available can depend on whether you choose
  an active/passive or active/active configuration.

Active/Passive vs. Active/Active
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Stateful services may be configured as active/passive or active/active:

:term:`active/passive configuration`
  Maintains a redundant instance
  that can be brought online when the active service fails.
  For example, OpenStack writes to the main database
  while maintaining a disaster recovery database that can be brought online
  if the main database fails.

  A typical active/passive installation for a stateful service maintains
  a replacement resource that can be brought online when required.
  Requests are handled using a :term:`virtual IP address (VIP)` that
  facilitates returning to service with minimal reconfiguration.
  A separate application (such as Pacemaker or Corosync) monitors
  these services, bringing the backup online as necessary.

:term:`active/active configuration`
  Each service also has a backup but manages both the main and
  redundant systems concurrently.
  This way, if there is a failure, the user is unlikely to notice.
  The backup system is already online and takes on increased load
  while the main system is fixed and brought back online.

  Typically, an active/active installation for a stateless service
  maintains a redundant instance, and requests are load balanced using
  a virtual IP address and a load balancer such as HAProxy.

  A typical active/active installation for a stateful service includes
  redundant services, with all instances having an identical state. In
  other words, updates to one instance of a database update all other
  instances. This way a request to one instance is the same as a
  request to any other. A load balancer manages the traffic to these
  systems, ensuring that operational systems always handle the
  request.

Clusters and quorums
~~~~~~~~~~~~~~~~~~~~

The quorum specifies the minimal number of nodes
that must be functional in a cluster of redundant nodes
in order for the cluster to remain functional.
When one node fails and failover transfers control to other nodes,
the system must ensure that data and processes remain sane.
To determine this, the contents of the remaining nodes are compared
and, if there are discrepancies, a "majority rules" algorithm is implemented.

For this reason, each cluster in a high availability environment should
have an odd number of nodes and the quorum is defined as more than a half
of the nodes.
If multiple nodes fail so that the cluster size falls below the quorum
value, the cluster itself fails.

For example, in a seven-node cluster, the quorum should be set to
floor(7/2) + 1 == 4. If quorum is four and four nodes fail simultaneously,
the cluster itself would fail, whereas it would continue to function, if
no more than three nodes fail. If split to partitions of three and four nodes
respectively, the quorum of four nodes would continue to operate the majority
partition and stop or fence the minority one (depending on the
no-quorum-policy cluster configuration).

And the quorum could also have been set to three, just as a configuration
example.

.. note::

  Note that setting the quorum to a value less than floor(n/2) + 1 is not
  recommended and would likely cause a split-brain in a face of network
  partitions.

Then, for the given example when four nodes fail simultaneously,
the cluster would continue to function as well. But if split to partitions of
three and four nodes respectively, the quorum of three would have made both
sides to attempt to fence the other and host resources. And without fencing
enabled, it would go straight to running two copies of each resource.

This is why setting the quorum to a value less than floor(n/2) + 1 is
dangerous. However it may be required for some specific cases, like a
temporary measure at a point it is known with 100% certainty that the other
nodes are down.

When configuring an OpenStack environment for study or demonstration purposes,
it is possible to turn off the quorum checking;
this is discussed later in this guide.
Production systems should always run with quorum enabled.


Single-controller high availability mode
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpenStack supports a single-controller high availability mode
that is managed by the services that manage highly available environments
but is not actually highly available because
no redundant controllers are configured to use for failover.
This environment can be used for study and demonstration
but is not appropriate for a production environment.

It is possible to add controllers to such an environment
to convert it into a truly highly available environment.


High availability is not for every user. It presents some challenges.
High availability may be too complex for databases or
systems with large amounts of data. Replication can slow large systems
down. Different setups have different prerequisites. Read the guidelines
for each setup.

High availability is turned off as the default in OpenStack setups.
