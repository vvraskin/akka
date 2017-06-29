# Cluster across multiple data centers

@@@ note

Cluster teams are a work-in-progress feature, and behavior is still expected to change.

@@@

This chapter describes how @ref:[Akka Cluster](cluster-usage.md) can be used across
multiple data centers, availability zones or regions. 

The features are based on that nodes can be assigned to a team by setting the 
`akka.cluster.team` configuration property. A node can only belong to one team
and if no team is specified a node will belong to the `default` team.

The grouping of nodes into teams is not limited to the physical boundaries of data centers,
even though that is the primary use case. It could also be used as a logical grouping
for other reasons, such as isolation of certain nodes to improve stability or splitting
up a large cluster into smaller groups of nodes for better scalability. 

## Membership

Some @ref:[membership transitions](common/cluster.md#membership-lifecycle) are managed by 
one node called the @ref:[leader](common/cluster.md#leader). There is one leader per team and
it is responsible for these transitions for the members within the same team. Members of
other teams are managed independently by other team leaders. These actions cannot be performed
while there are any unreachability observations among the nodes in the team, but unreachability
across different teams don't influence the progress of membership management within a team.
Nodes can be added and removed also when there are network partitions between data centers,
which is impossible if nodes are not grouped into teams.

![cluster-team.png](../images/cluster-team.png)

User actions like joining, leaving, and downing can be sent to any node in the cluster,
not only to the nodes in the team of the node. Seed nodes are also global.

The team membership is implemented by adding the team name prefixed with `"team-"` to the 
@ref:[roles](cluster-usage.md#node-roles) of the member and thereby this information is known
by all other members in the cluster.

## Failure Detection

@ref:[Failure detection](cluster-usage.md#failure-detector) is performed by sending heartbeat messages
to detect if a node is unreachable. This is done more frequently and with more certainty among
the nodes in the same team than across teams. The failure detection across different teams should
be interpreted as an indication of problem with the network link between the data centers.

Two different failure detectors can be configured for these two purposes:

* `akka.cluster.failure-detector` for for failure detection within own team
* `akka.cluster.inter-team-failure-detector` for for failure detection across different teams

When @ref:[subscribing to cluster events](cluster-usage.md#cluster-subscriber) the `UnreachableMember` and
`ReachableMember` events are for observations within the own team. The same team as where the
subscription was registered.

For cross team unreachability notifications you can subscribe to `UnreachableTeam` and `ReachbleTeam`
events.

## Cluster Singleton

The [Cluster Singleton](cluster-singleton.md) is a singleton per team. If you start the 
`ClusterSingletonManager` on all nodes and you have defined 3 different teams there will be
3 active singleton instances in the cluster, one in each team. This is taken care of automatically, 
but is important to be aware of.

The reason why the singleton is per team and not global is that there is not enough consistency
in the membership management when using one leader per team to select a single global singleton.

If you need a global singleton you have to pick one team to host that singleton and only start the
`ClusterSingletonManager` on nodes of that team. If the team is unavailable from another team the
singleton is unavailable, which is a reasonable trade-off when selecting a design for consistency
over availability.

The `ClusterSingletonProxy` is by default routing messages to the singleton in the own team, but
it can be started with a `team` parameter in the `ClusterSingletonProxySettings` to define that 
it should route messages to a singleton located in another team. That is useful for example when
having a global singleton in one team and accessing it from other teams.

## Cluster Sharding

The coordinator in [Cluster Sharding](cluster-sharding.md) is a Cluster Singleton and therefore,
as explained above, Cluster Sharding is also per team. Each team will have its own coordinator
and regions, isolated from other teams. If you start an entity type with the same name on all 
nodes and you have defined 3 different teams and then send messages to the same entity id to
sharding regions in all teams you will end up with 3 active entity instances for that entity id,
one in each team. 

Especially when used together with Akka Persistence that is based on the single-writer principle
it is important to avoid running the same entity at multiple locations at the same time with a
shared data store. Lightbend's Akka Team is working on solution that will support multiple active
entities that will work nicely together with Cluster Sharding across multiple data centers.

If you need global entities you have to pick one team to host that entity type and only start
`ClusterSharding` on nodes of that team. If the team is unavailable from another team the
entities are unavailable, which is a reasonable trade-off when selecting design for consistency
over availability.

The Cluster Sharding proxy is by default routing messages to the shard regions in the own team, but
it can be started with a `team` parameter to define that it should route messages to a singleton
located in another team. That is useful for example when having global entities in one team and 
accessing them from other teams.

Another way to manage global entities is to make sure that certain entity ids are located in 
only one team by routing the messages to the right region. For example, the routing function
could be that odd entity ids are routed to data center A and even entity ids to data center B.
Before sending the message to the local region actor you make the decision of which team it should
be routed to. Messages for another team can be sent with a sharding proxy as explained above and
messages for the own team are sent to the local region.
