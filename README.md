# WaterTable - VMs Flowing to the Lowest Point

**Note: This is purely an architecture design at the moment, no code**

The core architecture of WaterTable is a set of KVM hosts, each with an agent (WaterTable) 
running on them.

The agent has a number of features:

  1) It reports (when queried by other nodes) it's own host's system statistics. Most simply, 
     we'll use CPU and memory free
  2) If thresholds are met locally, it attempts to migrate VMs running on it's own node to 
     other nodes (ie, a push architecture). If another node is a better place to run the VM
     attempting migration, the migration is started.
  3) It is aware of other nodes in the cluster in a peer-to-peer fashion. All nodes query 
     all other nodes to ascertain workload suitability.

## Additional Features

**Draining** - 
In addition to auto-balancing, there would also be a function to allow a node to be set to
a 'draining' state which would move any migratable VMs off the host within the next minutes 
(or perhaps tens of minutes on a busy host), and of course stop any incoming requests for
migration on to the host. This could be used most obviously for node maintance. When a node 
is no longer running any workloads, it can be safely removed. After a node is returned to 
service and `draining=false` set, workloads would begin migrating back from peer nodes, 
due to it having far more free resources than the other nodes.

With this functionality, zero-downtime node removals are not only possible, but nearly automated,
with three steps:

 * draining = true. Wait.
 * [your required node maintenance]
 * draining = false. Wait.

**Unmigratable VMs** - 
There would be functionality to set given VMs as not-migratable (for example, guest OSes 
running critical workloads which can not handle even the brief interruption of a migration).

VMs could also be set as unmigratable during given time allocations (for example, business hours),
allowing them to be migrated when slightly downtime can be tolerated off hours, or in a 
maintance window.

**Anti-Affinity** - 
Some VMs should be intentionally running on different nodes to provide redundancy. For example,
it's entirely possible that a 4-vm-cluster running a low-utilization application end up 
running all it's VMs on a single node due to migration policies. As such, functionality would 
be provided to avoid running given VMs alongside other selected VMs (perhaps with a tagging 
functionality) unless absolutely required.


## Concepts

**Peer-to-peer agent queries** -
Queries are lightweight and would easily scale up to large clusters. If required, nodes could 
query busy nodes less often, since they are less likely to be a suitable migration target. 
However, it should scale up to clusters of 10-15 before this is even remotely a concern.

However... even that would likely not be needed for most scales.

Given a large KVM cluster of 50 nodes: 50 nodes attempting to all contact each 
other (say: once every 10 seconds) would still only result in each node handling 49 incoming 
queries. In particular if local expensive data is cached, 5 queries per second is 
absolutely trivial load for a likely-high-powered KVM hypervisor node.

Finally, given a large enough cluster, it could simply be broken up into two WaterTable clusters,
splitting hosts into two or more groups, and setting each group to only peer with the other hosts
in that group.

At some point in the future, perhaps these groups could be treated as singular logical nodes, 
and groups could less frequently peer with each other to allow singular cluster operation, but 
less peer-to-peer traffic between nodes.


**Run Favoribility Algorithm** - 
An algorithm will need to be written to score a given VMs against multiple hosts in a cluster,
to find where it should best be running (ie, where the most free resources are, or at the least, 
where it is the least-oversubscribed). This scoring would also be used to decide if migration
should occur at all. This score would need to be able to handle differently-specced nodes, so
it would need to take into account both the guest requirements, as well as the host resource
availibility (both allocated availibility and actual availibility, to allow smart oversubscribing
of quiet VMs).

For example, if a VM has a run score of 65 on the current host, and only 66 and 67 on other 
hosts, a migration is likely not worth it. However, if another host is found with a score of 93, 
migration to this host is a clear choice to balance cluster load.
